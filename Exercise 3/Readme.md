# Egzersiz 3 - TCPdump

Bu egzersizde **TCPdump** paket analizörünü (packet analyzer) fuzz'layacağız. Amaç, TCPdump 4.9.2 sürümünde [**CVE-2017-13028**](https://www.cvedetails.com/cve/CVE-2017-13028/) için bir çökme/PoC bulmaktır.

<details>
  <summary>CVE-2017-13028 güvenlik açığı hakkında daha fazla bilgi için, buraya tıklayın!</summary>
  --------------------------------------------------------------------------------------------------------
  
**CVE-2017-13028**, bir BOOTP (Bootstrap Protocol) paketi yoluyla tetiklenebilen bir Sınır Dışı Okuma (Out-of-bounds Read) güvenlik açığıdır.
  
  Sınır Dışı Okuma (Out-of-bounds Read), program veriyi hedeflenen tamponun (buffer) sonundan öteye ya da başlangıcından öncesine doğru okuduğunda meydana gelen bir güvenlik açığıdır.
  
  Sonuç olarak, uzaktan saldırganlara hizmet reddine (denial of service) yol açma ya da işlem belleğinden potansiyel olarak hassas bilgiler edinme imkânı verir.
  
  Sınır Dışı Okuma (Out-of-bounds Read) güvenlik açıkları hakkında daha fazla bilgiyi şu bağlantıda bulabilirsiniz: https://cwe.mitre.org/data/definitions/125.html
  
</details>

## Neler öğreneceksiniz?
Bu egzersizi tamamladığınızda şunları bilmiş olacaksınız:
- **ASan (Address Sanitizer)** nedir, çalışma zamanı (runtime) bellek hatası tespit aracı
- Bir hedefi fuzz'lamak için ASan nasıl kullanılır
- Çökmeleri ASan ile incelemek (triage) ne kadar kolaydır

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
1) TCPdump'ı fuzz'lamak için verimli bir yol bulmak
2) Fuzzing için ASan'ı nasıl etkinleştireceğinizi bulmaya çalışmak
3) Birkaç benzersiz çökme elde edene kadar TCPdump'ı fuzz'lamak
4) Güvenlik açığı için PoC bulmak amacıyla çökmeleri incelemek (triage)
5) Sorunu düzeltmek

**Tahmini süre = 4 saat**

---------------------------------------------------------------------------------------------------------------------------------------------------

<details>
  <summary>SPOİLER UYARISI! : Çözüm içeride</summary>

### Hedefinizi indirin ve derleyin

Önce fuzzing hedefimizi alalım. Fuzz'lamak istediğiniz proje için yeni bir dizin oluşturun:
```
cd $HOME
mkdir fuzzing_tcpdump && cd fuzzing_tcpdump/
```

tcpdump-4.9.2.tar.gz'yi indirin ve sıkıştırmasını açın
```
wget https://github.com/the-tcpdump-group/tcpdump/archive/refs/tags/tcpdump-4.9.2.tar.gz
tar -xzvf tcpdump-4.9.2.tar.gz
```

Ayrıca TCPdump için gerekli olan platformlar arası bir kütüphane olan libpcap'ı indirmemiz gerekiyor. libpcap-1.8.0.tar.gz'yi indirin ve sıkıştırmasını açın:
```
wget https://github.com/the-tcpdump-group/libpcap/archive/refs/tags/libpcap-1.8.0.tar.gz
tar -xzvf libpcap-1.8.0.tar.gz
```

``libpcap-libpcap-1.8.0`` ismini ``libpcap-1.8.0`` olarak değiştirmemiz gerekiyor. Aksi takdirde tcpdump, ``libpcap.a`` yerel yolunu bulamaz:
```
mv libpcap-libpcap-1.8.0/ libpcap-1.8.0
```
  
libpcap'ı derleyip kurun:

```
cd $HOME/fuzzing_tcpdump/libpcap-1.8.0/
./configure --enable-shared=no
make
```

Şimdi, tcpdump'ı derleyip kurabiliriz:
```
cd $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/
./configure --prefix="$HOME/fuzzing_tcpdump/install/"
make
make install
```

Her şeyin düzgün çalıştığını test etmek için sadece şunu yazın:
```
$HOME/fuzzing_tcpdump/install/sbin/tcpdump -h
```
  
ve şuna benzer bir şey görmelisiniz

![](Images/Image1.png)
  
**Bir sonraki adıma geçmeden önce, sürüm numaralarınızın yukarıdaki numaralarla eşleştiğini kontrol edin**

### Seed corpus (tohum külliyatı) oluşturma

"./tests" klasöründe çok sayıda .pcap örneği bulabilirsiniz. Bu .pcap dosyalarını şu komut satırıyla çalıştırabilirsiniz:
```
$HOME/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r [.pcap dosyası]
```
  
Örneğin:
```
$HOME/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r ./tests/geneve.pcap
```
  
Ve çıktı şöyle görünmeli
![](Images/Image2.png)
  
### AddressSanitizer
  
**AddressSanitizer (ASan)**, C ve C++ için hızlı bir bellek hatası tespit edicidir. Başlangıçta Google tarafından geliştirildi (Konstantin Serebryany, Derek Bruening, Alexander Potapenko, Dmitry Vyukov) ve ilk kez Mayıs 2011'de yayımlandı.

Bir derleyici enstrümantasyon modülü ve bir çalışma zamanı (runtime) kütüphanesinden oluşur. Araç, heap, stack ve global nesnelere sınır dışı (out-of-bounds) erişimleri, ayrıca use-after-free, double-free ve bellek sızıntısı (memory leak) hatalarını bulabilmektedir.

AddressSanitizer açık kaynaklıdır ve 3.1 sürümünden itibaren LLVM derleyici araç zinciriyle bütünleşiktir. Başlangıçta bir LLVM projesi olarak geliştirilmiş olsa da GCC'ye taşınmıştır ve GCC >= 4.8 sürümlerine dahildir.

AddressSanitizer hakkında daha fazla bilgiyi şu [bağlantıda](https://clang.llvm.org/docs/AddressSanitizer.html) bulabilirsiniz.
  
### ASan etkinleştirilmiş derleme
  
Şimdi tcpdump'ı (ve libpcap'ı) ASAN etkinleştirilmiş şekilde derleyeceğiz.

Her şeyden önce, önceden derlenmiş tüm nesne dosyalarını ve çalıştırılabilir dosyaları temizleyeceğiz:
```
rm -r $HOME/fuzzing_tcpdump/install
cd $HOME/fuzzing_tcpdump/libpcap-1.8.0/
make clean

cd $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/
make clean
```

Şimdi, ``configure`` ve ``make`` çağırmadan önce `AFL_USE_ASAN=1` ayarlıyoruz:

```
cd $HOME/fuzzing_tcpdump/libpcap-1.8.0/
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/fuzzing_tcpdump/install/"
AFL_USE_ASAN=1 make

cd $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/
AFL_USE_ASAN=1 CC=afl-clang-lto ./configure --prefix="$HOME/fuzzing_tcpdump/install/"
AFL_USE_ASAN=1 make
AFL_USE_ASAN=1 make install
```  
  
**Afl-clang-lto derlemesi tamamlanması birkaç dakika sürebilir**  
  
### Fuzzing zamanı

Şimdi, fuzzer'ı şu komutla çalıştırabilirsiniz:
```
afl-fuzz -m none -i $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/tests/ -o $HOME/fuzzing_tcpdump/out/ -s 123 -- $HOME/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r @@
```

**Not: 64-bit sistemlerde ASAN çok fazla sanal bellek talep eder. Bu yüzden AFL'daki bellek sınırlarını devre dışı bırakan "-m none" seçeneğini ayarladım**
  
Bir süre sonra, birden fazla çökmeniz olmalı:
  
![](Images/Image3.png)


### İnceleme (Triage)
  
ASan ile derlenmiş bir programı hata ayıklamak, önceki egzersizlere göre çok daha kolaydır. Yapmanız gereken tek şey programı çökme dosyasıyla beslemektir:
  
```
$HOME/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r '/home/antonio/fuzzing_tcpdump/out/default/crashes/id:000000,sig:06,src:002318+001583,time:10357087,op:splice,rep:8'
```

ve yürütme izini (execution trace) de içeren, çökmenin güzel bir özetini alacaksınız:
  
![](Images/Image4.png)  

### Sorunu düzeltin

Egzersizin son adımı hatayı düzeltmektir! Düzeltmeden sonra hedefinizi yeniden derleyin ve PoC'nuzun artık programı çökertmediğini kontrol edin. Bu son kısım, öğrenciye (size) egzersiz olarak bırakılmıştır.
  
  <details>
  <summary>Çözüm içeride</summary>
   --------------------------------------------------------------------------------------------------
    
  Resmî düzeltme:
  - https://github.com/the-tcpdump-group/tcpdump/commit/85078eeaf4bf8fcdc14a4e79b516f92b6ab520fc#diff-05f854a9033643de07f0d0059bc5b98f3b314eeb1e2499ea1057e925e6501ae8L381
    
   </details> 

Alternatif olarak, TCPdump'ın daha yeni bir sürümünü indirebilir ve her iki hatanın da düzeltildiğini kontrol edebilirsiniz.
  
</details>
