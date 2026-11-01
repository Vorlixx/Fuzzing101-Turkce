# Egzersiz 6 - GIMP

Bu egzersizde **GIMP** görsel düzenleyicisini fuzz'layacağız. Amaç, GIMP 2.8.16 sürümünde [**CVE-2016-4994**](https://www.cvedetails.com/cve/CVE-2016-4994/) için bir çökme/PoC bulmaktır.

<details>
  <summary>CVE-2016-4994 güvenlik açığı hakkında daha fazla bilgi için, buraya tıklayın!</summary>
  --------------------------------------------------------------------------------------------------------
  
  **CVE-2016-4994**, özel olarak hazırlanmış (crafted) bir XCF dosyası yoluyla tetiklenebilen bir Use-After-Free (Serbest Bırakıldıktan Sonra Kullanma) güvenlik açığıdır.
  
  Use after free hataları, bir program serbest bırakıldıktan (freed) sonra bir işaretçiyi (pointer) kullanmaya devam ettiğinde meydana gelir.
  
  Bu, geçerli verinin bozulmasından keyfi kodun çalıştırılmasına kadar çok sayıda olumsuz sonuca yol açabilir.
 
  Use-After-Free güvenlik açıkları hakkında daha fazla bilgiyi şu bağlantıda bulabilirsiniz: https://cwe.mitre.org/data/definitions/416.html
  
</details>


## Neler öğreneceksiniz?
Bu egzersizi tamamladığınızda şunları bilmiş olacaksınız:
- Fuzzing'i hızlandırmak için persistent (kalıcı) modun nasıl kullanılacağını
- Etkileşimli / GUI uygulamalarının nasıl fuzz'lanacağını

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

## Persistent (Kalıcı) mod

AFL persistent modu, **in-process fuzzer'lar** temeline dayanır: tek bir process kullanan, hedef process'e kod enjekte eden ve girdi değerlerini bellek içinde değiştiren fuzzer'lardır.

afl-fuzz, in-process fuzzing'in avantajlarını daha geleneksel bir çoklu-process aracının sağlamlığıyla birleştiren bir çalışma modunu destekler: **Persistent Mod**.

Persistent modda, AFL++ her bir fuzz yürütmesi için yeni bir process fork'lamak yerine, tek bir fork'lanmış process içinde hedefi birden çok kez fuzz'lar. Bu mod, fuzzing hızını 20 kata kadar artırabilir.

Hedefin temel yapısı şöyle olurdu:
```C
  //Program initialization

  while (__AFL_LOOP(10000)) {

    /* Read input data. */
    /* Call library code to be fuzzed. */
    /* Reset state. */
  }
  
  //End of fuzzing
  
```

AFL++ persistent modu hakkında daha fazla bilgiyi şurada bulabilirsiniz: https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.persistent_mode.md

## Kendin yap!
Bu egzersizi tamamlamak için şunları yapmanız gerekir:
1) AFL persistent modunu etkinleştirmek için GIMP kaynak kodunu değiştirmenin verimli bir yolunu bulmak
2) XCF örneklerinden oluşan bir seed corpus (tohum külliyatı) oluşturmak
3) İsteğe bağlı: XCF dosya formatı için bir fuzzing sözlüğü (dictionary) oluşturmak
4) GIMP'i birkaç benzersiz çökme elde edene kadar fuzz'lamak. Mümkün olduğunca çok AFL örneği (CPU çekirdeği) kullanmanızı öneririm
5) Güvenlik açığı için PoC bulmak amacıyla çökmeleri incelemek (triage)
6) Sorunları düzeltmek
7) Bonus: Diğer küçük hatalardan bazılarını bulmak

**Tahmini süre = 7 saat**

---------------------------------------------------------------------------------------------------------------------------------------------------

<details>
  <summary>SPOİLER UYARISI! : Çözüm içeride</summary>

### Hedefinizi indirin ve derleyin
  
Önce fuzzing hedefimizi alalım. Fuzz'lamak istediğiniz proje için yeni bir dizin oluşturun:
```
cd $HOME
mkdir Fuzzing_gimp && cd Fuzzing_gimp
```
  
Şimdi, gerekli bağımlılıkları kurun:
```
sudo apt-get install build-essential libatk1.0-dev libfontconfig1-dev libcairo2-dev libgudev-1.0-0 libdbus-1-dev libdbus-glib-1-dev libexif-dev libxfixes-dev libgtk2.0-dev python2.7-dev libpango1.0-dev libglib2.0-dev zlib1g-dev intltool libbabl-dev
```

Ayrıca **GEGL 0.2 (Generic Graphics Library)**'ye de ihtiyacımız var. Ne yazık ki, Ubuntu dağıtımımızda gegl-0.2.0 paketini bulamıyoruz. Yani bu kütüphaneyi kaynak koddan indirip derlememiz gerekiyor. Sadece şunu yazın:
```
wget https://download.gimp.org/pub/gegl/0.2/gegl-0.2.0.tar.bz2
tar xvf gegl-0.2.0.tar.bz2 && cd gegl-0.2.0
```
  
Şimdi, kaynak kodda iki küçük değişiklik yapmamız gerekiyor:
```
sed -i 's/CODEC_CAP_TRUNCATED/AV_CODEC_CAP_TRUNCATED/g' ./operations/external/ff-load.c
sed -i 's/CODEC_FLAG_TRUNCATED/AV_CODEC_FLAG_TRUNCATED/g' ./operations/external/ff-load.c
```

Gegl-0.2'yi derleyip kurun:  
```
./configure --enable-debug --disable-glibtest  --without-vala --without-cairo --without-pango --without-pangocairo --without-gdk-pixbuf --without-lensfun --without-libjpeg --without-libpng --without-librsvg --without-openexr --without-sdl --without-libopenraw --without-jasper --without-graphviz --without-lua --without-libavformat --without-libv4l --without-libspiro --without-exiv2 --without-umfpack
make -j$(nproc)
sudo make install
```
Test aşamasında bazı hata mesajları görürseniz endişelenmeyin.

Şimdi, GIMP 2.8.16'yı indirip sıkıştırmasını açıyoruz:
```
cd ..
wget https://mirror.klaus-uwe.me/gimp/pub/gimp/v2.8/gimp-2.8.16.tar.bz2
tar xvf gimp-2.8.16.tar.bz2 && cd gimp-2.8.16/
```
  
GIMP'i **afl-clang-lto** derleyicisiyle derleme zamanı (biraz sürebilir): 
```
CC=afl-clang-lto CXX=afl-clang-lto++ PKG_CONFIG_PATH=$PKG_CONFIG_PATH:$HOME/Fuzzing_gimp/gegl-0.2.0/ CFLAGS="-fsanitize=address" CXXFLAGS="-fsanitize=address" LDFLAGS="-fsanitize=address" ./configure --disable-gtktest --disable-glibtest --disable-alsatest --disable-nls --without-libtiff --without-libjpeg --without-bzip2 --without-gs --without-libpng --without-libmng --without-libexif --without-aa --without-libxpm --without-webkit --without-librsvg --without-print --without-poppler --without-cairo-pdf --without-gvfs --without-libcurl --without-wmf --without-libjasper --without-alsa --without-gudev --disable-python --enable-gimp-console --without-mac-twain --without-script-fu --without-gudev --without-dbus --disable-mp --without-linux-input --without-xvfb-run --with-gif-compression=none --without-xmc --with-shm=none --enable-debug  --prefix="$HOME/Fuzzing_gimp/gimp-2.8.16/install"
make -j$(nproc)
make install
```
  
### Persistent (Kalıcı) mod
  
İki çok basit yaklaşım var:
  
- İlki ``app.c`` dosyasını değiştirmek ve AFL_LOOP makrosunu for döngüsüne dahil etmektir:
  
![](Images/Image0.png)
  
- İkincisi AFL_LOOP makrosunu ``xcf_load_invoker`` fonksiyonunun içine koymaktır:
  
![](Images/Image1.png)
  
İlki bize farklı girdi formatlarını hedefleme imkânı verirken, ikincisi daha hızlıdır ve hatayı yakalama şansımız daha fazla olur.
  
İkincisinin yamasını (patch) [buradan](./persistent.patch) indirebilirsiniz
  
### Seed corpus (tohum külliyatı) oluşturma
  
Birden çok .xcf örneği elde etmek için birden fazla GIMP projesi oluşturmanızı ve kaydetmenizi öneririm  
  
Alternatif olarak, sadece [SampleInput.xcf](./SampleInput.xcf) dosyasını AFL girdi klasörünüze kopyalayabilirsiniz
  
### Fuzzing zamanı

Güvenlik açığı GIMP çekirdeğini (core) etkilediğinden, ihtiyacımız olmayan eklentileri kaldırarak biraz başlatma süresi kazandırabiliriz:
```
rm ./install/lib/gimp/2.0/plug-ins/*
```
  
Şimdi, fuzzer'ı şu komutla çalıştırabilirsiniz:
```
ASAN_OPTIONS=detect_leaks=0,abort_on_error=1,symbolize=0 afl-fuzz -i './afl_in' -o './afl_out' -D -t 100 -- ./install/bin/gimp-console-2.8 --verbose -d -f @@
```
  
Bazı notlar:
- `gimp-console-2.8`, GIMP'in sadece konsol sürümüdür
- Deterministik mutasyonları (-D) etkinleştirmenizi öneririm
- Kodda bir sonsuz döngü hatası da var, bu yüzden düşük bir zaman aşımı sınırı (ex. -t 100) ayarlamamız gerekiyor. Bu zaman aşımı sınırı makinenizin kapasitesine bağlıdır, bu yüzden kendi özel durumunuza göre ayarlamanız gerekecek.

  
Bir süre sonra, birden fazla çökmeniz olmalı:
![](Images/Image2.png)  
  
### İnceleme (Triage)
  
ASan izi (trace) şöyle görünebilir:
  
![](Images/Image3.png)  
  
### Sorunları düzeltme
  
Egzersizin son adımı her iki hatayı da düzeltmektir. Düzeltmelerden sonra hedefinizi yeniden derleyin ve PoC'larınızın artık programı çökertmediğini kontrol edin. Bu son kısım, öğrenciye (size) egzersiz olarak bırakılmıştır.
  
  <details>
  <summary>Çözüm içeride</summary>
   --------------------------------------------------------------------------------------------------
    
  Resmî düzeltme:
    
  - https://gitlab.gnome.org/GNOME/gimp/-/commit/6d804bf9ae77bc86a0a97f9b944a129844df9395
    
   </details> 

Alternatif olarak, GIMP'in daha yeni bir sürümünü indirebilir ve her iki hatanın da düzeltildiğini kontrol edebilirsiniz.
  
  
### Bonus

Kodda başka küçük hatalar da var. Ben şunları buldum:

- Bir null referansı (null dereference) hatası
- Bir sonsuz döngü hatası
- Bir bellek tükenmesi (memory exhaustion) hatası

Hepsini bulmayı deneyebilmenizi öneririm!
  
</details>
