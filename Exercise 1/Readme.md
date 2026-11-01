# Egzersiz 1 - Xpdf

Bu egzersizde **Xpdf** PDF görüntüleyicisini fuzz'layacağız. Amaç, XPDF 3.02 sürümünde [**CVE-2019-13288**](https://www.cvedetails.com/cve/CVE-2019-13288/) için bir çökme (crash)/PoC bulmaktır.

<details>
  <summary>CVE-2019-13288 güvenlik açığı hakkında daha fazla bilgi için, buraya tıklayın!</summary>
  --------------------------------------------------------------------------------------------------------
  
  **CVE-2019-13288**, özel olarak hazırlanmış (crafted) bir dosya yoluyla sonsuz özyinelemeye (infinite recursion) yol açabilen bir güvenlik açığıdır.
  
  Bir programdaki her çağrılan fonksiyon, stack üzerinde bir yığın çerçevesi (stack frame) ayırdığı için, bir fonksiyon çok kez özyinelemeli olarak çağrılırsa stack belleğinin tükenmesine ve programın çökmesine yol açabilir.
  
  Sonuç olarak, uzaktan bir saldırgan bunu bir DoS saldırısı için kullanabilir.
 
  Kontrolsüz Özyineleme (Uncontrolled Recursion) güvenlik açıkları hakkında daha fazla bilgiyi aşağıdaki bağlantıda bulabilirsiniz: https://cwe.mitre.org/data/definitions/674.html
  
</details>

## Neler öğreneceksiniz?
Bu egzersizi tamamladıktan sonra, AFL ile fuzzing'in temellerini bilmiş olacaksınız:   
- Bir hedef uygulamayı enstrümantasyonla (instrumentation) derlemek
- Bir fuzzer çalıştırmak (afl-fuzz)
- Çökmeleri bir hata ayıklayıcı ile incelemek (triage) (GDB)

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

## Hedefinizi indirin ve derleyin

Önce fuzzing hedefimizi alalım. Fuzz'lamak istediğiniz proje için yeni bir dizin oluşturun:
```
cd $HOME
mkdir fuzzing_xpdf && cd fuzzing_xpdf/
```
Ortamınızı tamamen hazır hale getirmek için bazı ek araçları kurmanız gerekebilir (make ve gcc)
```
sudo apt install build-essential
```

Xpdf 3.02'yi indirin:
```
wget https://dl.xpdfreader.com/old/xpdf-3.02.tar.gz
tar -xvzf xpdf-3.02.tar.gz
```

Xpdf'i derleyin:
```
cd xpdf-3.02
sudo apt update && sudo apt install -y build-essential gcc
./configure --prefix="$HOME/fuzzing_xpdf/install/"
make
make install
```

Derlemeyi test etme zamanı. Her şeyden önce, birkaç PDF örneği indirmeniz gerekecek:
```
cd $HOME/fuzzing_xpdf
mkdir pdf_examples && cd pdf_examples
wget https://github.com/mozilla/pdf.js-sample-files/raw/master/helloworld.pdf
wget http://www.africau.edu/images/default/sample.pdf
wget https://www.melbpc.org.au/wp-content/uploads/2017/10/small-example-pdf-file.pdf
```

Şimdi, pdfinfo ikili dosyasını (binary) şununla test edebiliriz:
```
$HOME/fuzzing_xpdf/install/bin/pdfinfo -box -meta $HOME/fuzzing_xpdf/pdf_examples/helloworld.pdf
```

Şuna benzer bir şey görmelisiniz:

![](Images/Image2.png)

## AFL++ Kurulumu
Bu kursta, [AFL++ fuzzer](https://github.com/AFLplusplus/AFLplusplus)'ın en son sürümünü kullanacağız.

Her şeyi iki şekilde kurabilirsiniz:  

<details>
  <summary>Yerel kurulum (önerilen seçenek)</summary>
  
  Bağımlılıkları kurun

  ```
  sudo apt-get update
  sudo apt-get install -y build-essential python3-dev automake git flex bison libglib2.0-dev libpixman-1-dev python3-setuptools
  sudo apt-get install -y lld-11 llvm-11 llvm-11-dev clang-11 || sudo apt-get install -y lld llvm llvm-dev clang 
  sudo apt-get install -y gcc-$(gcc --version|head -n1|sed 's/.* //'|sed 's/\..*//')-plugin-dev libstdc++-$(gcc --version|head -n1|sed 's/.* //'|sed 's/\..*//')-dev
  ```

  AFL++'ı indirin ve derleyin
  ```
  cd $HOME
  git clone https://github.com/AFLplusplus/AFLplusplus && cd AFLplusplus
  export LLVM_CONFIG="llvm-config-11"
  make distrib
  sudo make install
  ```
</details>

<details>
  <summary>Docker imajı</summary>
  
  Docker'ı kurun
  ```
  sudo apt install docker
  ```
  
  İmajı çekin (pull)
  ```
  docker pull aflplusplus/aflplusplus
  ```
  
  AFLPlusPlus docker container'ını başlatın:
  ```
  docker run -ti -v $HOME:/home aflplusplus/aflplusplus
  ```
  ve ardından şunu yazın
  ```
  export $HOME="/home"
  ```
</details>

Şimdi her şey yolunda gittiyse, afl-fuzz'ı çalıştırabilmelisiniz. Sadece şunu yazın

```afl-fuzz```

ve şuna benzer bir şey görmelisiniz:

![](Images/Image1.png)

### AFL++ ile tanışın

AFL bir **coverage-guided fuzzer**'dır, yani yeni yürütme yollarını (execution paths) ve potansiyel hataları keşfetmek amacıyla her bir mutasyona uğramış girdi için kapsam (coverage) bilgisi toplar. Kaynak kodu mevcut olduğunda AFL enstrümantasyon kullanabilir; her basic block'un (fonksiyonlar, döngüler vb.) başına fonksiyon çağrıları ekler.

Hedef uygulamamız için enstrümantasyonu etkinleştirmek üzere, kodu AFL'ın derleyicileri ile derlememiz gerekir.

Her şeyden önce, önceden derlenmiş tüm nesne dosyalarını ve çalıştırılabilir dosyaları temizleyeceğiz:
```
rm -r $HOME/fuzzing_xpdf/install
cd $HOME/fuzzing_xpdf/xpdf-3.02/
make clean
```

Ve şimdi xpdf'i **afl-clang-fast** derleyicisini kullanarak derleyeceğiz:
```
export LLVM_CONFIG="llvm-config-11"
CC=$HOME/AFLplusplus/afl-clang-fast CXX=$HOME/AFLplusplus/afl-clang-fast++ ./configure --prefix="$HOME/fuzzing_xpdf/install/"
make
make install
```

Şimdi fuzzer'ı şu komutla çalıştırabilirsiniz:
```
afl-fuzz -i $HOME/fuzzing_xpdf/pdf_examples/ -o $HOME/fuzzing_xpdf/out/ -s 123 -- $HOME/fuzzing_xpdf/install/bin/pdftotext @@ $HOME/fuzzing_xpdf/output
```

Her bir seçeneğin kısa açıklaması:
- *-i* girdi dosyalarını (dosya örnekleri olarak da bilinir) koyacağımız dizini belirtir
- *-o* AFL++'ın mutasyona uğramış dosyaları saklayacağı dizini belirtir
- *-s* kullanılacak sabit rastgele tohum değerini (seed) belirtir
- *@@* hedefin komut satırındaki yer tutucudur (placeholder) ve AFL bunu her girdi dosyasının adıyla değiştirir

Yani temelde, fuzzer her farklı girdi dosyası için  
`$HOME/fuzzing_xpdf/install/bin/pdftotext <girdi-dosya-adi> $HOME/fuzzing_xpdf/output` komutunu çalıştıracaktır.

*"Hmm, your system is configured to send core dump notifications to an external utility..."* gibi bir mesaj alırsanız, sadece şunu yapın:
```
sudo su
echo core >/proc/sys/kernel/core_pattern
exit
```

Birkaç dakika sonra şuna benzer bir şey görmelisiniz:

![](Images/Image3.png)

Benzersiz çökmelerin (uniq. crashes) sayısını gösteren kırmızı renkli **"uniq. crashes"** değerini görebilirsiniz. Bu çökme dosyalarını `$HOME/fuzzing_xpdf/out/` dizininde bulabilirsiniz. İlk çökme bulunduktan sonra fuzzer'ı durdurabilirsiniz; üzerinde çalışacağımız çökme budur. Bir çökme elde etmeniz makinenizin performansına bağlı olarak bir veya iki saate kadar sürebilir.

Bu aşamada şunları öğrendiniz: 
- Enstrümantasyonlu bir AFL derleyicisi kullanarak hedefi nasıl derleyeceğinizi
- afl++'ı nasıl başlatacağınızı
- Hedefinizin benzersiz çökmelerini nasıl tespit edeceğinizi

Peki şimdi ne olacak? Bu hata hakkında herhangi bir bilgimiz yok, sadece programın bir çökmesi var... Hata ayıklama ve inceleme (debugging & triage) zamanı!

## Kendin yap!
Bu egzersizi tamamlamak için şunları yapmanız gerekir:
1) İlgili dosya ile çökmeyi yeniden üretmek
2) Sorunu bulmak için çökmeyi hata ayıklamak
3) Sorunu düzeltmek

**Tahmini süre = 120 dakika**

---

<details>
  <summary>SPOİLER UYARISI! : Çözüm içeride</summary>
  
### Çökmeyi yeniden üretin

`$HOME/fuzzing_xpdf/out/` dizininde çökmeye karşılık gelen dosyayı bulun. Dosya adı şuna benzer: `id:000000,sig:11,src:001504+000002,time:924544,op:splice,rep:16`

![](Images/Image3_1.png)

Bu dosyayı pdftotext binary'sine girdi olarak verin
```
$HOME/fuzzing_xpdf/install/bin/pdftotext '$HOME/fuzzing_xpdf/out/default/crashes/<dosya_adiniz>' $HOME/fuzzing_xpdf/output
```
Bu bir segmentation fault'a (bellek erişim hatası) yol açacak ve programın çökmesiyle sonuçlanacaktır.

![](Images/Image3_2.png)

### İnceleme (Triage)

Programın bu girdiyle neden çöktüğünü anlamak için gdb kullanın.

- **GDB hakkında iyi ve kısa bir giriş için http://people.cs.pitt.edu/~mosse/gdb-note.html adresine göz atabilirsiniz**
  
Her şeyden önce, sembolik (symbolic) bir stack trace elde etmek için Xpdf'i hata ayıklama (debug) bilgileriyle yeniden derlemeniz gerekir:

```
rm -r $HOME/fuzzing_xpdf/install
cd $HOME/fuzzing_xpdf/xpdf-3.02/
make clean
CFLAGS="-g -O0" CXXFLAGS="-g -O0" ./configure --prefix="$HOME/fuzzing_xpdf/install/"
make
make install
```

Şimdi GDB'yi çalıştırabilirsiniz:

```
gdb --args $HOME/fuzzing_xpdf/install/bin/pdftotext $HOME/fuzzing_xpdf/out/default/crashes/<dosya_adiniz> $HOME/fuzzing_xpdf/output
```
Ardından GDB içinde şunu yazın:
```
 >> run
```

Her şey yolunda gittiyse şu çıktıyı görmelisiniz:

![](Images/Image4.png)

Ardından backtrace (çağrı izi) almak için ```bt``` yazın:

![](Images/Image5.png)

Çağrı yığınını (call stack) kaydırırsanız, sonsuz özyinelemeye işaret ediyor gibi görünen "Parser::getObj" metoduna ait birçok çağrı göreceksiniz. https://www.cvedetails.com/cve/CVE-2019-13288/ adresine giderseniz, açıklamanın GDB'den aldığımız backtrace ile eşleştiğini görebilirsiniz.
  
### Sorunu düzeltin

Egzersizin son adımı hatayı düzeltmektir! Düzeltmeden sonra hedefinizi yeniden derleyin ve kullanım senaryonuzunuzun artık bir segmentation fault'a yol açmadığını kontrol edin. Bu son kısım, öğrenciye (size) egzersiz olarak bırakılmıştır.

Alternatif olarak, hatanın zaten düzeltildiği Xpdf 4.02'yi indirebilir ve segmentation fault'un ortadan kalktığını kontrol edebilirsiniz.

</details>
