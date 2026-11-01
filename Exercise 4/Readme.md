# Egzersiz 4 - LibTIFF

Bu kez **LibTIFF** görsel kütüphanesini fuzz'layacağız. Amaç, libtiff 4.0.4 sürümünde [**CVE-2016-9297**](https://www.cvedetails.com/cve/CVE-2016-9297/) için bir çökme/PoC bulmak ve çökmenizin/PoC'nuzun **kod kapsamı (code coverage) verisini ölçmektir**.

<details>
  <summary>CVE-2016-9297 güvenlik açığı hakkında daha fazla bilgi için, buraya tıklayın!</summary>
  --------------------------------------------------------------------------------------------------------
  
**CVE-2016-9297**, özel olarak hazırlanmış TIFF_SETGET_C16ASCII veya TIFF_SETGET_C32_ASCII etiket değerleri yoluyla tetiklenebilen bir Sınır Dışı Okuma (Out-of-bounds Read) güvenlik açığıdır.
  
  Sınır Dışı Okuma (Out-of-bounds Read), program veriyi hedeflenen tamponun (buffer) sonundan öteye ya da başlangıcından öncesine doğru okuduğunda meydana gelen bir güvenlik açığıdır.
  
  Sonuç olarak, uzaktan saldırganlara hizmet reddine (denial of service) yol açma ya da işlem belleğinden potansiyel olarak hassas bilgiler edinme imkânı verir.
  
  Sınır Dışı Okuma (Out-of-bounds Read) güvenlik açıkları hakkında daha fazla bilgiyi şu bağlantıda bulabilirsiniz: https://cwe.mitre.org/data/definitions/125.html
  
</details>

## Neler öğreneceksiniz?
Bu egzersizi tamamladığınızda şunları bilmiş olacaksınız:
- LCOV kullanarak kod kapsamını (code coverage) nasıl ölçeceğinizi
- Fuzzing'in etkinliğini artırmak için kod kapsamı verisini nasıl kullanacağınızı

## Başlamadan Önce Okuyun
- Egzersizi, çözüme bakmadan **kendiniz çözmenizi** öneririm. Gücünüzün yettiği kadar çabalayın ve ancak takılırsanız aşağıdaki örnek çözüme bakın.
- AFL, deterministik olmayan (non-deterministic) bir test algoritması kullanır, bu yüzden iki fuzzing oturumu asla aynı olmaz. Bu yüzden **sabit bir tohum değeri (seed) ayarlamanızı (-s 123)** şiddetle tavsiye ederim. Bu sayede fuzzing sonuçlarınız burada gösterilenlere benzer olacak ve egzersizleri daha kolay takip etmenizi sağlayacak.  
- Yeni bir güvenlik açığı bulursanız, **lütfen projeye bir güvenlik raporu gönderin**. Bu süreç hakkında yardıma ihtiyacınız olursa ya da herhangi bir şüpheniz olursa [GitHub Security Lab](mailto:securitylab.github.com) size yardımcı olabilir :)

## İletişim
Takıldınız mı ve yardım mı arıyorsunuz? Bu kursu daha iyi hale getirmek için önerileriniz mi var, yoksa daha fazla fuzzing içeriği oluşturmamız için sadece olumlu geri bildirimler mi? 
Fuzzing deneyiminizi toplulukla paylaşmak ister misiniz?
GitHub Security Lab Slack'ine katılın ve `#fuzzing` kanalına gidin. [GitHub Security Lab Slack daveti isteyin](mailto:securitylab-social@github.com?subject=Request%20an%20invite%20to%20the%20GitHub%20Security%20Lab%20Slack)

## Ortam

Tüm egzersizler **Ubuntu 20.04.2 LTS** üzerinde test edilmiştir. Farklı fuzzing sonuçlarını önlemek için **aynı işletim sistemi sürümünü** kullanmanızı ve en iyi performans için AFL++'ı **bare-metal** (sanallaştırılmamış) donanımda, sanallaştırılmış makinelerde değil, çalıştırmanızı şiddetle tavsiye ederim.

Aksi takdirde, Ubuntu 20.04.2 LTS VMware imajını [burada](https://drive.google.com/file/d/1_m1x-SHcm7Muov2mlmbbt8nkrMYp0Q3K/view?usp=sharing) bulabilirsiniz. VMware yerine VirtualBox da kullanabilirsiniz.

Bu sanal makinenin kullanıcı adı / şifresi `fuzz` / `fuzz`'tur.

## Kendin yap!
Bu egzersizi tamamlamak için şunları yapmanız gerekir:
1) LibTiff'i (ASan etkin şekilde) birkaç benzersiz çökme elde edene kadar fuzz'lamak
2) Güvenlik açığı için PoC bulmak amacıyla çökmeleri incelemek (triage)
3) Bu PoC'nun kod kapsamını ölçmek
4) Sorunu düzeltmek

**Tahmini süre = 3 saat**

---------------------------------------------------------------------------------------------------------------------------------------------------

<details>
  <summary>SPOİLER UYARISI! : Çözüm içeride</summary>

### Hedefinizi indirin ve derleyin

Önce fuzzing hedefimizi alalım. Fuzz'lamak istediğiniz proje için yeni bir dizin oluşturun:
```
cd $HOME
mkdir fuzzing_tiff && cd fuzzing_tiff/
```

libtiff 4.0.4'ü indirin ve sıkıştırmasını açın:
```
wget https://download.osgeo.org/libtiff/tiff-4.0.4.tar.gz
tar -xzvf tiff-4.0.4.tar.gz
```


Şimdi, libtiff'i derleyip kurabiliriz:
```
cd tiff-4.0.4/
./configure --prefix="$HOME/fuzzing_tiff/install/" --disable-shared
make
make install
```
  
Hedef binary olarak sadece ``/bin`` klasöründe bulunan ``tiffinfo`` binary'sini fuzz'layabiliriz. Seed girdi külliyatı olarak ``/test/images/`` klasöründeki örnek görselleri kullanacağız.
  
Her şeyin düzgün çalıştığını test etmek için sadece şunu yazın:
```
$HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w $HOME/fuzzing_tiff/tiff-4.0.4/test/images/palette-1c-1b.tiff
```
  
ve şuna benzer bir şey görmelisiniz
![](Images/Image1.png)
  
Son komut satırında tüm şu bayrakları etkinleştirdim: "-j -c -r -s -w". Bu, **kod kapsamını (code coverage)** iyileştirmek ve hatayı bulma şansını artırmak içindir.

Peki ama belirli bir girdi dosyasının kod kapsamını nasıl ölçebiliriz?
  
### Kod kapsamı (Code coverage)
  
Kod kapsamı, her bir kod satırının kaç kez tetiklendiğini gösteren bir yazılım metriğidir. Kod kapsamını kullanarak fuzzer'ın kodun hangi kısımlarına ulaştığını öğrenecek ve fuzzing sürecini görselleştireceğiz.

Her şeyden önce, lcov'u kurmamız gerekir. Bunu şu komutla yapabiliriz:
```
sudo apt install lcov
```

Şimdi libTIFF'i ``--coverage`` bayrağıyla (derleyici ve bağlayıcı/linker) yeniden derlememiz gerekiyor:
```
rm -r $HOME/fuzzing_tiff/install
cd $HOME/fuzzing_tiff/tiff-4.0.4/
make clean
  
CFLAGS="--coverage" LDFLAGS="--coverage" ./configure --prefix="$HOME/fuzzing_tiff/install/" --disable-shared
make
make install
```  
Ardından kod kapsamı verisini şunu yazarak toplayabiliriz:

```
cd $HOME/fuzzing_tiff/tiff-4.0.4/
lcov --zerocounters --directory ./
lcov --capture --initial --directory ./ --output-file app.info
$HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w $HOME/fuzzing_tiff/tiff-4.0.4/test/images/palette-1c-1b.tiff
lcov --no-checksum --directory ./ --capture --output-file app2.info
```
  
Her bir komutu açıklamaya çalışacağım:
  - ``lcov --zerocounters --directory ./`` : Önceki sayaçları sıfırla
  - ``lcov --capture --initial --directory ./ --output-file app.info`` : Enstrümante edilmiş her satır için sıfır kapsam içeren "baseline" (temel) kapsam veri dosyasını döndürür
  - ``$HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w $HOME/fuzzing_tiff/tiff-4.0.4/test/images/palette-1c-1b.tiff`` : Analiz etmek istediğiniz uygulamayı çalıştırın. Farklı girdilerle birden çok kez çalıştırabilirsiniz
  - ``lcov --no-checksum --directory ./ --capture --output-file app2.info``: Mevcut kapsam durumunu app2.info dosyasına kaydedin

Son olarak, HTML çıktısını üretmemiz gerekir:
```
genhtml --highlight --legend -output-directory ./html-coverage/ ./app2.info
```
  
Her şey yolunda gittiyse, kod kapsamı raporu ``html-coverage`` klasöründe oluşturuldu. Sadece ``./html-coverage/index.html`` dosyasını açın ve şuna benzer bir şey görmelisiniz:
  
![](Images/Image2_2.png)
  
Şimdi farklı klasör ve dosyalara göz atabilir ve her satırın kaç kez çalıştırıldığını görebilirsiniz.
  
### Fuzzing zamanı

Şimdi libtiff'i ASAN etkinleştirilmiş şekilde derleyeceğiz.

Her şeyden önce, önceden derlenmiş tüm nesne dosyalarını ve çalıştırılabilir dosyaları temizleyeceğiz:
```
rm -r $HOME/fuzzing_tiff/install
cd $HOME/fuzzing_tiff/tiff-4.0.4/
make clean
```

Şimdi, make çağırmadan önce AFL_USE_ASAN=1 ayarlıyoruz:
```
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --prefix="$HOME/fuzzing_tiff/install/" --disable-shared
AFL_USE_ASAN=1 make -j4
AFL_USE_ASAN=1 make install
```

Şimdi, fuzzer'ı şu komutla çalıştırabilirsiniz:
```
afl-fuzz -m none -i $HOME/fuzzing_tiff/tiff-4.0.4/test/images/ -o $HOME/fuzzing_tiff/out/ -s 123 -- $HOME/fuzzing_tiff/install/bin/tiffinfo -D -j -c -r -s -w @@
```

Birkaç dakika sonra şuna benzer bir şey görmelisiniz:
![](Images/Image2.png)
  
### İnceleme (Triage)
  
ASan izi (trace) şöyle görünebilir:
  
![](Images/Image3.png)
  
### Kod kapsamı ölçümü
  
Şimdi, PoC'nuzun kod kapsamını ölçmeye çalışın. Bu kısmı tamamlamak için **bir kapsam HTML raporu elde etmeniz gerekiyor**, yukarıdaki örneğe benzer şekilde.
  
### Sorunu düzeltin

Egzersizin son adımı hatayı düzeltmektir! Düzeltmeden sonra hedefinizi yeniden derleyin ve PoC'nuzun artık programı çökertmediğini kontrol edin. Bu son kısım, öğrenciye (size) egzersiz olarak bırakılmıştır.
  
  <details>
  <summary>Çözüm içeride</summary>
   --------------------------------------------------------------------------------------------------
    
  Resmî düzeltme:
  - https://github.com/vadz/libtiff/commit/30c9234c7fd0dd5e8b1e83ad44370c875a0270ed
    
   </details> 

Alternatif olarak, LibTIFF'in daha yeni bir sürümünü indirebilir ve her iki hatanın da düzeltildiğini kontrol edebilirsiniz.
  
  
</details>
