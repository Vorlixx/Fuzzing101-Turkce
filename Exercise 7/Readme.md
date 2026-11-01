# Egzersiz 7 - VLC Media Player

Bu egzersizde **VLC** medya oynatıcısını fuzz'layacağız. Amaç, VLC 3.0.7.1 sürümünde [**CVE-2019-14776**](https://nvd.nist.gov/vuln/detail/CVE-2019-14776) için bir çökme/PoC bulmaktır.

<details>
  <summary>CVE-2019-14776 güvenlik açığı hakkında daha fazla bilgi için, buraya tıklayın!</summary>
  --------------------------------------------------------------------------------------------------------
  
  **CVE-2019-14776**, özel olarak hazırlanmış (crafted) bir WMV/ASF (Windows Media Video) dosyası yoluyla tetiklenebilen bir Sınır Dışı Okuma (Out-of-bounds Read) güvenlik açığıdır.
  
  Sınır Dışı Okuma (Out-of-bounds Read), program veriyi hedeflenen tamponun (buffer) sonundan öteye ya da başlangıcından öncesine doğru okuduğunda meydana gelen bir güvenlik açığıdır.

Sonuç olarak, uzaktan saldırganlara hizmet reddine (denial of service) yol açma ya da işlem belleğinden potansiyel olarak hassas bilgiler edinme imkânı verir.

Sınır Dışı Okuma (Out-of-bounds Read) güvenlik açıkları hakkında daha fazla bilgiyi şu bağlantıda bulabilirsiniz: https://cwe.mitre.org/data/definitions/125.html
  
</details>


## Neler öğreneceksiniz?
Bu egzersizi tamamladığınızda şunları bilmiş olacaksınız:
- Programın yalnızca ilgili kısımlarını enstrümante etmek için Kısmi Enstrümantasyonu (Partial Instrumentation) nasıl kullanacağınızı
- Büyük uygulamaları daha verimli test etmek için fuzzing harness'leri nasıl yazacağınızı

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

## Kısmi Enstrümantasyon (Partial Instrumentation)

Evolutionary bir coverage-guided fuzzer kullanmanın avantajlarından biri, yeni yürütme yollarını (execution paths) kendi başına bulabilmesidir. Ancak bu, genellikle bir dezavantaj da olabilir. Bu, özellikle her modülün belirli bir görevi yaptığı, son derece modüler bir mimariye sahip yazılımlarla (VLC media player durumunda olduğu gibi) karşılaştığımızda geçerlidir.

Diyelim ki fuzzer'a geçerli bir MKV dosyası verdik. Ancak girdi dosyasının birkaç mutasyonundan sonra, dosyanın "magic bytes" değerleri değişti ve artık girdi dosyası programımız tarafından bir AVI dosyası olarak görülüyor. Bu nedenle, bu "mutasyona uğramış MKV dosyası" artık AVI Demux tarafından işleniyor. Bir süre sonra, dosyanın magic bytes'ları bir kez daha değişiyor ve artık dosya bir MPEG dosyası olarak görülüyor. Her iki durumda da, bu yeni mutasyona uğramış dosyanın kod kapsamımızı artırma potansiyeli çok düşüktür, çünkü bu yeni dosya herhangi bir geçerli sözdizimi yapısına (syntax structure) sahip olmayacaktır.

Kısacası, kod kapsamına kısıtlama koymazsak, fuzzer kolayca yanlış bir yol seçebilir ve bu da fuzzing sürecini daha az etkili hale getirir.

Bu sorunu çözmek için AFL++, hangi fonksiyonların/dosyaların enstrümantasyonlu ya da enstrümantasyonsuz derleneceğini belirtmenize olanak tanıyan bir Kısmi Enstrümantasyon (Partial Instrumentation) özelliği içerir. Bu, fuzzer'ın programın önemli kısımlarına odaklanmasına yardımcı olur ve ilginç olmayan kod yollarını çalıştırarak oluşan istenmeyen gürültü ve bozulmayı önler.

Bunu kullanmak için, derleme yaparken AFL_LLVM_ALLOWLIST ortam değişkenini ayarlıyoruz. Bu ortam değişkeni, enstrümante edilmesi gereken tüm fonksiyonların/dosya adlarını içeren bir dosyaya işaret etmelidir.

AFL++ kısmi enstrümantasyonu hakkında daha fazla bilgiyi şurada bulabilirsiniz: https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.instrument_list.md

## Kendin yap!
Bu egzersizi tamamlamak için şunları yapmanız gerekir:
1) ASF demuxing fonksiyonlarını ve dosya adlarını içeren bir kısmi enstrümantasyon (partial instrumentation) dosyası yazmak
2) ASF dosya formatını fuzz'lamak için bir fuzzing harness yazmak (persistent mod)
3) ASF örneklerinden oluşan bir seed corpus (tohum külliyatı) oluşturmak
4) İsteğe bağlı: ASF dosya formatı için bir fuzzing sözlüğü (dictionary) oluşturmak
5) VLC'yi birkaç benzersiz çökme elde edene kadar fuzz'lamak. Mümkün olduğunca çok AFL örneği (CPU çekirdeği) kullanmanızı öneririm
6) Güvenlik açığı için PoC bulmak amacıyla çökmeleri incelemek (triage)
7) Sorunları düzeltmek

**Tahmini süre = 6 saat**

---------------------------------------------------------------------------------------------------------------------------------------------------

<details>
  <summary>SPOİLER UYARISI! : Çözüm içeride</summary>

### Hedefi indirin
  
Önce fuzzing hedefimizi alalım. Fuzz'lamak istediğiniz proje için yeni bir dizin oluşturun:
```
cd $HOME
mkdir fuzzing_vlc && cd fuzzing_vlc
```
  
vlc-3.0.7.1.tar.xz'yi indirin ve sıkıştırmasını açın:
```
wget https://download.videolan.org/pub/videolan/vlc/3.0.7.1/vlc-3.0.7.1.tar.xz
tar -xvf vlc-3.0.7.1.tar.xz && cd vlc-3.0.7.1/
```
    
### VLC'yi derleyin:
```
./configure --prefix="$HOME/fuzzing_vlc/vlc-3.0.7.1/install" --disable-a52 --disable-lua --disable-qt
make -j$(nproc)
```

Her şeyin düzgün çalıştığını test etmek için sadece şunu yazın:
```
./bin/vlc-static --help
```
  
ve şuna benzer bir şey görmelisiniz

![](Images/image0.png)
  

### Seed corpus (tohum külliyatı) oluşturma
 
[ffmpeg samples reposunda](https://samples.ffmpeg.org/) birçok video örneği bulabilirsiniz
  
Bazı örnekler seçmenizi ve ardından bir video düzenleyici kullanarak video dosyasını mümkün olan en küçük boyuta indirmenizi öneririm.
  
Bunlar, açık kaynaklı bazı video düzenleyicilerin örnekleridir:
  
- [OpenShot](https://www.openshot.org/)
- [Shotcut](https://shotcut.org/)
  
Ya da daha kolay bir yol olarak, sadece [short2.wmv](./InputCorpus/short2.wmv) ve [veryshort.wmv](./InputCorpus/veryshort.wmv) dosyalarını AFL girdi klasörünüze kopyalayın.
  
  
### Fuzzing harness
  
Doğrudan ``vlc-static`` binary'sini fuzz'lamayı denerseniz, AFL'ın saniyede yalnızca birkaç yürütme aldığını görürsünüz. Bunun nedeni, VLC'nin başlatılmasının (startup) çok zaman alıcı olmasıdır. Bu yüzden VLC'yi fuzz'lamak için **özel bir fuzzing harness** oluşturmanızı öneririm.
  
Fuzzing harness'imi dahil etmek için ``./test/vlc-demux-run.c`` dosyasını değiştirmeyi seçtim. Bu sayede, harness'i sadece şunu yaparak derleyebilirsiniz:
  
```
cd test
make vlc-demux-run -j$(nproc) LDFLAGS="-fsanitize=address"
cd ..  
```
  
Hata ASF demuxing'de bulunduğundan, ``vlc_demux_process_memory`` fonksiyonunu çağırıyorum. Bu fonksiyon, önceden bellekte saklanmış bir veri tamponunu (data buffer) demux etmeye çalışır. Kod değişikliklerimi [burada](./fuzzing_harness.patch) bulabilirsiniz
  
  
### Kısmi enstrümantasyon
  
Başlangıçta sadece ASF demuxing'de rol oynayan dosya adlarını dahil etmeyi denedim. Ne yazık ki, bu yaklaşım işe yaramadı.

Dosya adlarıyla eşleştirme [her zaman mümkün değil](https://github.com/AFLplusplus/AFLplusplus/issues/1018#issuecomment-879045408) gibi görünüyor, bu yüzden fonksiyon eşleştirme ve dosya adı eşleştirmeyi içeren karma bir yaklaşımı tercih ettim:
  

![](Images/image2.png)  
  
Kısmi enstrümantasyon dosyamı [buradan](./Partial_instrumentation) indirebilirsiniz

  
### Küçük değişiklikler
  
ASF fuzzing hızını artırmak için, ``modules/demux/libasf.c`` dosyasına bu yamayı (patch) uygulamanızı öneririm (şimdilik daha fazla ipucu yok ;) ) : [speedup.patch](./speedup.patch)
  
### Fuzzing zamanı  
  
VLC'yi **afl-clang-fast** derleyicisiyle ve ASAN etkinleştirilmiş şekilde derleme zamanı:
  
```
CC="afl-clang-fast" CXX="afl-clang-fast++" ./configure --prefix="$HOME/fuzzing_vlc/vlc-3.0.7.1/install" --disable-a52 --disable-lua --disable-qt --with-sanitizer=address
AFL_LLVM_ALLOWLIST=$HOME/fuzzing_vlc/vlc-3.0.7.1/Partial_instrumentation make -j$(nproc) LDFLAGS="-fsanitize=address"
```
  
Fuzzing harness'i derleyin:
```
cd test
make vlc-demux-run -j$(nproc) LDFLAGS="-fsanitize=address"
cd ..  
```

Şimdi, fuzzer'ı şu komutla çalıştırabilirsiniz:
```
afl-fuzz -t 100 -m none -i './afl_in' -o './afl_out' -x asf_dictionary.dict -D -M master -- ./test/vlc-demux-run @@  
  
```
  
Bazı notlar:
- Zaman aşımı (timeout) parametresi büyük ölçüde bilgisayara bağlıdır. Bu değeri ayarlamanız gerekecek.

  
Bir süre sonra, birden fazla çökmeniz olmalı:
![](Images/image1.png)  

### İnceleme (Triage)
  
ASan izi (trace) şöyle görünebilir:
  
![](Images/image3.png)  
  
### Sorunları düzeltme
  
Egzersizin son adımı hatayı düzeltmektir. Düzeltmeden sonra hedefinizi yeniden derleyin ve PoC'nuzun artık programı çökertmediğini kontrol edin. Bu son kısım, öğrenciye (size) egzersiz olarak bırakılmıştır.
  
  <details>
  <summary>Çözüm içeride</summary>
   --------------------------------------------------------------------------------------------------
    
  Resmî düzeltme:
    
  - https://github.com/videolan/vlc/commit/fdbdd677c1e6262f31771b0ba10afb24aabf108c#diff-a22170125046390274dd33c2cb5bb0e99d485e6708b376f40978de9534ac55a9
    
   </details> 

Alternatif olarak, VLC'nin daha yeni bir sürümünü indirebilir ve her iki hatanın da düzeltildiğini kontrol edebilirsiniz.
  
