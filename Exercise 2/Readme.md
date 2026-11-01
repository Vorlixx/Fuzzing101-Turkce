# Egzersiz 2 - libexif

Bu kez **libexif** EXIF ayrıştırma kütüphanesini fuzz'layacağız. Amaç, libexif 0.6.14 sürümünde [**CVE-2009-3895**](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3895) için bir çökme/PoC ve [**CVE-2012-2836**](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2012-2836) için başka bir çökme bulmaktır.

<details>
  <summary>CVE-2009-3895 ve CVE-2012-2836 güvenlik açıkları hakkında daha fazla bilgi için, buraya tıklayın!</summary>
  --------------------------------------------------------------------------------------------------------
  
**CVE-2009-3895**, geçersiz bir EXIF görseli ile tetiklenebilen heap tabanlı bir tampon taşması (heap-based buffer overflow) durur.
  
  Heap tabanlı tampon taşması, heap veri alanında meydana gelen bir tampon taşması türüdür ve genellikle açık dinamik bellek yönetimiyle (malloc() ve free() fonksiyonlarıyla bellek ayırma/ayırma iptali) ilgilidir.
  
  Sonuç olarak, uzaktan bir saldırgan, bu sorunu etkilenen kütüphaneyi kullanan bir uygulamanın bağlamında (context) keyfi kod çalıştırmak için kullanabilir.
  
  Heap tabanlı tampon taşması (Heap-based buffer overflow) güvenlik açıkları hakkında daha fazla bilgiyi şu bağlantıda bulabilirsiniz: https://cwe.mitre.org/data/definitions/122.html
  
  -------------------------------------------------------------------------
  
  **CVE-2012-2836**, özel olarak hazırlanmış EXIF etiketleri (tags) içeren bir görsel yoluyla tetiklenebilen bir Sınır Dışı Okuma (Out-of-bounds Read) güvenlik açığıdır.
  
  Sınır Dışı Okuma (Out-of-bounds Read), program veriyi hedeflenen tamponun (buffer) sonundan öteye ya da başlangıcından öncesine doğru okuduğunda meydana gelen bir güvenlik açığıdır.
  
  Sonuç olarak, uzaktan saldırganlara hizmet reddine (denial of service) yol açma ya da işlem belleğinden potansiyel olarak hassas bilgiler edinme imkânı verir.
  
  Sınır Dışı Okuma (Out-of-bounds Read) güvenlik açıkları hakkında daha fazla bilgiyi şu bağlantıda bulabilirsiniz: https://cwe.mitre.org/data/definitions/125.html
  
</details>
 

## Neler öğreneceksiniz?
Bu egzersizi tamamladığınızda şunları nasıl yapacağınızı bilmiş olacaksınız:
- Harici (dış) bir uygulama kullanarak bir kütüphaneyi fuzz'lamak
- *afl-clang-fast*'tan daha hızlı olan ve daha iyi sonuçlar sağlayan, çakışmasız (collision-free) bir enstrümantasyon olan **afl-clang-lto**'yu kullanmak
- İnceleme (triage) için GDB konsoluna kolay bir alternatif olarak Eclipse IDE kullanmak

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
1) libexif kütüphanesini kullanan bir arayüz uygulaması bulmak
2) Exif örneklerinden oluşan bir seed corpus (tohum külliyatı) oluşturmak
3) libexif'i ve seçilen uygulamayı afl-clang-lto ile fuzz'lanacak şekilde derlemek
4) Birkaç benzersiz çökme elde edene kadar libexif'i fuzz'lamak
5) Her bir güvenlik açığı için PoC bulmak amacıyla çökmeleri incelemek (triage)
6) Sorunları düzeltmek

**Tahmini süre = 6 saat**

---

<details>
  <summary>SPOİLER UYARISI! : Çözüm içeride</summary>

### Hedefinizi indirin ve derleyin

Önce fuzzing hedefimizi alalım. Fuzz'lamak istediğimiz proje için yeni bir dizin oluşturun:
```
cd $HOME
mkdir fuzzing_libexif && cd fuzzing_libexif/
```

libexif-0.6.14'ü indirin ve sıkıştırmayı açın:
```
wget https://github.com/libexif/libexif/archive/refs/tags/libexif-0_6_14-release.tar.gz
tar -xzvf libexif-0_6_14-release.tar.gz
```
  
libexif'i derleyip kurun:

```
cd libexif-libexif-0_6_14-release/
sudo apt-get install autopoint libtool gettext libpopt-dev
autoreconf -fvi
./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/"
make
make install
```

### Bir arayüz uygulaması seçme
  
libexif bir kütüphane olduğundan, bu kütüphaneyi kullanan ve fuzz'lanacak başka bir uygulamaya ihtiyacımız olacak. Bu görev için **exif komut satırı aracını** kullanacağız. exif komut satırı aracı 0.6.15'i indirmek ve sıkıştırmasını açmak için şunu yazın:
```
cd $HOME/fuzzing_libexif
wget https://github.com/libexif/exif/archive/refs/tags/exif-0_6_15-release.tar.gz
tar -xzvf exif-0_6_15-release.tar.gz
```

Şimdi, exif komut satırı yardımcı programını derleyip kurabiliriz:
```
cd exif-exif-0_6_15-release/
autoreconf -fvi
./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/" PKG_CONFIG_PATH=$HOME/fuzzing_libexif/install/lib/pkgconfig
make
make install
```
  
Her şeyin düzgün çalıştığını test etmek için sadece şunu yazın:
```
$HOME/fuzzing_libexif/install/bin/exif
```
  
ve şuna benzer bir şey görmelisiniz

![](Images/Image1.png)

### Seed corpus (tohum külliyatı) oluşturma
  
Şimdi bazı exif örneklerine ihtiyacımız var. Şu repodaki örnek görselleri kullanacağız: https://github.com/ianare/exif-samples. Şununla indirebilirsiniz:
```
cd $HOME/fuzzing_libexif
wget https://github.com/ianare/exif-samples/archive/refs/heads/master.zip
unzip master.zip
```
  
Örnek olarak şunu yapabiliriz:
```
$HOME/fuzzing_libexif/install/bin/exif $HOME/fuzzing_libexif/exif-samples-master/jpg/Canon_40D_photoshop_import.jpg
```
  
Ve çıktı şöyle görünmeli
![](Images/Image2.png)

### Afl-clang-lto enstrümantasyonu

Şimdi libexif'i derleyici olarak **afl-clang-lto** kullanarak derleyeceğiz.

```
rm -r $HOME/fuzzing_libexif/install
cd $HOME/fuzzing_libexif/libexif-libexif-0_6_14-release/
make clean
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/"
make
make install
```

```
cd $HOME/fuzzing_libexif/exif-exif-0_6_15-release
make clean
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/" PKG_CONFIG_PATH=$HOME/fuzzing_libexif/install/lib/pkgconfig
make
make install
```

Gördüğünüz gibi, *afl-clang-fast* yerine **afl-clang-lto** kullandım. Genel olarak, *afl-clang-lto* mevcut en iyi seçenektir çünkü çakışmasız (collision-free) bir enstrümantasyondur ve *afl-clang-fast*'tan daha hızlıdır.

Ne zaman *afl-clang-lto* ne zaman *afl-clang-fast* kullanacağınızdan emin değilseniz, [AFLplusplus : instrumenting that target](https://github.com/AFLplusplus/AFLplusplus#1-instrumenting-that-target) sayfasından alınan şu şemaya bakabilirsiniz

```
+--------------------------------+
| clang/clang++ 11+ is available | --> use LTO mode (afl-clang-lto/afl-clang-lto++)
+--------------------------------+     see [instrumentation/README.lto.md](instrumentation/README.lto.md)
    |
    | if not, or if the target fails with LTO afl-clang-lto/++
    |
    v
+---------------------------------+
| clang/clang++ 6.0+ is available | --> use LLVM mode (afl-clang-fast/afl-clang-fast++)
+---------------------------------+     see [instrumentation/README.llvm.md](instrumentation/README.llvm.md)
    |
    | if not, or if the target fails with LLVM afl-clang-fast/++
    |
    v
 +--------------------------------+
 | gcc 5+ is available            | -> use GCC_PLUGIN mode (afl-gcc-fast/afl-g++-fast)
 +--------------------------------+    see [instrumentation/README.gcc_plugin.md](instrumentation/README.gcc_plugin.md) and
                                       [instrumentation/README.instrument_list.md](instrumentation/README.instrument_list.md)
    |
    | if not, or if you do not have a gcc with plugin support
    |
    v
   use GCC mode (afl-gcc/afl-g++) (or afl-clang/afl-clang++ for clang)
```
  
### Fuzzing zamanı

Şimdi, fuzzer'ı şu komutla çalıştırabilirsiniz:
```
afl-fuzz -i $HOME/fuzzing_libexif/exif-samples-master/jpg/ -o $HOME/fuzzing_libexif/out/ -s 123 -- $HOME/fuzzing_libexif/install/bin/exif @@
```
  
Birkaç dakika sonra, birden fazla çökmeniz olmalı:

![](Images/Image3.png)

### İnceleme (Triage)
  
#### Eclipse kurulumu
  
1. egzersizde çökmeleri incelemek (triage) için GDB konsolunu nasıl kullanacağımızı öğrendik. Bu ikinci egzersizde, hata ayıklama amaçları için Eclipse-CDT'yi nasıl kullanacağımızı göreceğiz.
  
Her şeyden önce, buradan indirebiliriz:
https://www.eclipse.org/downloads/download.php?file=/technology/epp/downloads/release/2021-03/R/eclipse-cpp-2021-03-R-linux-gtk-x86_64.tar.gz

JAVA-SDK sisteminizde kurulu değilse, Ubuntu'da şu komutla kurabilirsiniz:
```
sudo apt install default-jdk
```
  
Ardından sıkıştırmasını şöyle açabiliriz:
```
tar -xzvf eclipse-cpp-2021-03-R-linux-gtk-x86_64.tar.gz
```
  
Eclipse-CDT'yi başlattıktan sonra, kaynak kodumuzu Project Explorer'a aktarmamız (import) gerekiyor. Bunun için File -> Import'a gitmemiz ve ardından C/C++ -> "Existing code as makefile project" seçeneğini seçmemiz gerekir. Sonra "Linux GCC" seçip exif kaynak kodu klasörüne göz atmamız gerekir:
  
![](Images/Image4.png)
  
Her şey yolunda gittiyse, Project Explorer sekmesinde "exif" klasörünü görebilmelisiniz.
  
  
Şimdi hata ayıklama parametrelerini yapılandıracağız. Bunun için `Run -> Debug Configurations` yoluna gitmemiz gerekir. Sonra exif projemizi seçip exif binary'sine göz atmamız gerekir:
  
![](Images/Image5.png)  
  
Ardından girdi argümanlarını ayarlamamız gerekir. Bunun için `"Arguments"` sekmesine gidin ve AFL çökmelerinden birinin yolunu ayarlayın.
  
![](Images/Image6.png)
  
Son olarak, hata ayıklama oturumunu (debugging session) başlatmak için sadece `"Debug"` butonuna tıklamamız yeterli ve program main fonksiyonunun başında duracaktır.
  
Bu noktaya geldikten sonra, sadece `Run -> Resume` tıklamamız yeterli olup bir segmentation fault tespit edildiğinde yürütme duracaktır.
  
![](Images/Image7.png)  

### Sorunları düzeltme
  
Egzersizin son adımı her iki hatayı da düzeltmektir. Düzeltmelerden sonra hedefinizi yeniden derleyin ve PoC'larınızın artık programı çökertmediğini kontrol edin. Bu son kısım, öğrenciye (size) egzersiz olarak bırakılmıştır.
  
  <details>
  <summary>Çözüm içeride</summary>
   --------------------------------------------------------------------------------------------------
    
  Resmî düzeltmeler:
    
  - https://github.com/libexif/libexif/commit/8ce72b7f81e61ef69b7ad5bdfeff1516c90fa361
  - https://github.com/libexif/libexif/commit/00986f6fa979fe810b46e376a462c581f9746e06
    
   </details> 

Alternatif olarak, libexif'in daha yeni bir sürümünü indirebilir ve her iki hatanın da düzeltildiğini kontrol edebilirsiniz.
  
</details>
