# Egzersiz 5 - LibXML2

Bu egzersizde **LibXML2** XML ayrıştırma kütüphanesini fuzz'layacağız. Amaç, LibXML2 2.9.4 sürümünde [**CVE-2017-9048**](https://nvd.nist.gov/vuln/detail/CVE-2017-9048) için bir çökme/PoC bulmaktır.

<details>
  <summary>CVE-2017-9048 güvenlik açığı hakkında daha fazla bilgi için, buraya tıklayın!</summary>
  --------------------------------------------------------------------------------------------------------
  
  **CVE-2017-9048**, LibXML2'nin DTD doğrulama (validation) işlevselliğini etkileyen bir stack tampon taşması (stack buffer overflow) güvenlik açığıdır.
  
  Stack tampon taşması, üzerine yazılan tamponun (buffer) stack üzerinde ayrıldığı bir tampon taşması türüdür.
  
  Sonuç olarak, uzaktan bir saldırgan, bu sorunu etkilenen kütüphaneyi kullanan bir uygulamanın bağlamında (context) keyfi kod çalıştırmak için kullanabilir.

  Stack tampon taşması (stack buffer overflow) güvenlik açıkları hakkında daha fazla bilgiyi şu bağlantıda bulabilirsiniz: https://cwe.mitre.org/data/definitions/121.html
  
</details>

## Neler öğreneceksiniz?
Bu egzersizi tamamladığınızda şunları nasıl yapacağınızı bilmiş olacaksınız: 
- Fuzzer'ın yeni yürütme yolları bulmasına yardımcı olmak için özel (custom) sözlükler (dictionaries) kullanmak
- Fuzzing işini birden fazla çekirdeğe paralelleştirmek

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

## Sözlükler (Dictionaries)

Karmaşık metin tabanlı dosya formatlarını (XML gibi) fuzz'lamak istediğimizde, fuzzer'a temel sözdizimi (syntax) token'larının bir listesini içeren bir sözlük (dictionary) sağlamak faydalıdır.

AFL söz konusu olduğunda, böyle bir sözlük basitçe, AFL'ın bellek içi (in-memory) mevcut dosyaya değişiklikler uygulamak için kullandığı bir kelime veya değer kümesidir. Özellikle AFL, sözlükte sağlanan değerlerle şu değişiklikleri yapar:
- Override (Üzerine yazma): Belirli bir konumu n baytla değiştirir; n, sözlük girdisinin uzunluğudur.
- Insert (Ekleme): Sözlük girdisini dosyanın mevcut konumuna ekler, tüm karakterleri n konum aşağı iterek dosya boyutunu artırır.

Bol miktarda örneği [burada](https://github.com/AFLplusplus/AFLplusplus/tree/stable/dictionaries) bulabilirsiniz

## Paralelleştirme

Çok çekirdekli (multi-core) bir sisteminiz varsa, CPU kaynaklarınızdan en iyi şekilde yararlanmak için fuzzing işinizi paralelleştirmek iyi bir fikirdir.

### Bağımsız örnekler (Independent instances)

Bu, en basit paralelleştirme stratejisidir. Bu modda, tamamen ayrı afl-fuzz örnekleri çalıştırırız.

AFL'ın deterministik olmayan (non-deterministic) bir test algoritması kullandığını hatırlamak önemlidir. Yani birden çok AFL örneği çalıştırırsak, başarı şansımızı artırırız.

Bunun için, sadece birden fazla terminal penceresinde, her biri için farklı bir "çıktı klasörü" ayarlayarak birden fazla "afl-fuzz" örneği çalıştırmanız yeterlidir. Basit bir yaklaşım, sisteminizdeki çekirdek sayısı kadar fuzzing işi çalıştırmaktır.

Not: -s bayrağını kullanıyorsanız, her örnek için farklı bir tohum (seed) kullanmanız gerekir

### Paylaşımlı örnekler (Shared instances)

Paylaşımlı örneklerin kullanımı, paralel fuzzing için daha iyi bir yaklaşımdır. Bu durumda, her fuzzer örneği, diğer fuzzer'ların bulduğu tüm test dosyalarını (test cases) toplar.

Genellikle aynı anda yalnızca bir master (ana) örneğiniz olur:
```
./afl-fuzz -i afl_in -o afl_out -M Master -- ./program @@
```

ve N-1 adet slave (köle) örnek:
```
./afl-fuzz -i afl_in -o afl_out -S Slave1 -- ./program @@
./afl-fuzz -i afl_in -o afl_out -S Slave2 -- ./program @@
...
./afl-fuzz -i afl_in -o afl_out -S SlaveN -- ./program @@
```

## Kendin yap!
Bu egzersizi tamamlamak için şunları yapmanız gerekir:
1) LibXML2 kütüphanesini kullanan bir arayüz uygulaması bulmak
2) [SampleInput.xml](./SampleInput.xml) dosyasını AFL girdi klasörünüze kopyalamak
3) XML fuzz'lamak için özel bir sözlük (dictionary) oluşturmak
4) LibXML2'yi birkaç benzersiz çökme elde edene kadar fuzz'lamak. Mümkün olduğunca çok AFL örneği (CPU çekirdeği) kullanmanızı öneririm
5) Güvenlik açığı için PoC bulmak amacıyla çökmeleri incelemek (triage)
6) Sorunları düzeltmek

**Tahmini süre = 3 saat**


<details>
  <summary>SPOİLER UYARISI! : Çözüm içeride</summary>

### Hedefinizi indirin ve derleyin

Önce fuzzing hedefimizi alalım. Fuzz'lamak istediğiniz proje için yeni bir dizin oluşturun:
```
cd $HOME
mkdir Fuzzing_libxml2 && cd Fuzzing_libxml2
```

libxml2-2.9.4.tar.gz'yi indirin ve sıkıştırmasını açın
```
wget http://xmlsoft.org/download/libxml2-2.9.4.tar.gz
tar xvf libxml2-2.9.4.tar.gz && cd libxml2-2.9.4/
```

libxml2'yi derleyip kurun:
```
sudo apt-get install python-dev
CC=afl-clang-lto CXX=afl-clang-lto++ CFLAGS="-fsanitize=address" CXXFLAGS="-fsanitize=address" LDFLAGS="-fsanitize=address" ./configure --prefix="$HOME/Fuzzing_libxml2/libxml2-2.9.4/install" --disable-shared --without-debug --without-ftp --without-http --without-legacy --without-python LIBS='-ldl'
make -j$(nproc)
make install
```

Şimdi, her şeyin yolunda olduğunu şu komutla test edebiliriz:
```
./xmllint --memory ./test/wml.xml
```

ve şuna benzer bir şey görmelisiniz

![](Images/Image1.png)
  
### Seed corpus (tohum külliyatı) oluşturma

Her şeyden önce, bazı XML örneklerine ihtiyacımız var. Bu repoda sağlanan **SampleInput.xml** dosyasını kullanacağız:
```
mkdir afl_in && cd afl_in
wget https://raw.githubusercontent.com/Vorlixx/Fuzzing101-Turkce/main/Exercise%205/SampleInput.xml
cd ..
```
  
### Özel sözlük (Custom dictionary)

Şimdi, bir XML sözlüğü oluşturmanız gerekiyor. Alternatif olarak, AFL++ ile birlikte sağlanan XML sözlüğünü kullanabilirsiniz: 
```
mkdir dictionaries && cd dictionaries
wget https://raw.githubusercontent.com/AFLplusplus/AFLplusplus/stable/dictionaries/xml.dict
cd ..
```
### Fuzzing zamanı
  
Hatayı yakalamak için `--valid` parametresini etkinleştirmek zorunludur. Ayrıca sözlük yolunu **-x bayrağıyla** ayarladım ve deterministik mutasyonları **-D bayrağıyla** etkinleştirdim (sadece master fuzzer için):

Örneğin fuzzer'ı şu komutla çalıştırdım 
```
afl-fuzz -m none -i ./afl_in -o afl_out -s 123 -x ./dictionaries/xml.dict -D -M master -- ./xmllint --memory --noenc --nocdata --dtdattr --loaddtd --valid --xinclude @@
```

Başka bir slave örneği şununla çalıştırabilirsiniz:
```
afl-fuzz -m none -i ./afl_in -o afl_out -s 234 -S slave1 -- ./xmllint --memory --noenc --nocdata --dtdattr --loaddtd --valid --xinclude @@
```
  
**Komut satırı argümanlarını fuzz'lamak ilginizi çekiyor mu?** Şu [blog yazısına](https://securitylab.github.com/research/fuzzing-challenges-solutions-1/), "Fuzzing command-line arguments" bölümüne göz atın.

Bir süre sonra, birden fazla çökmeniz olmalı:
  
![](Images/Image2.png)

### İnceleme (Triage)
  
ASan ile derlenmiş bir programı hata ayıklamak, önceki egzersizlere göre çok daha kolaydır. Yapmanız gereken tek şey programı çökme dosyasıyla beslemektir:
  
```
./xmllint --memory --noenc --nocdata --dtdattr --loaddtd --valid --xinclude './afl_out/default/crashes/id:000000,sig:06,src:003963,time:12456489,op:havoc,rep:4'
```

ve yürütme izini (execution trace) de içeren, çökmenin güzel bir özetini alacaksınız:
  
![](Images/Image3.png)  
  
### Sorunu düzeltin

Egzersizin son adımı hatayı düzeltmektir! Düzeltmeden sonra hedefinizi yeniden derleyin ve PoC'nuzun artık programı çökertmediğini kontrol edin. Bu son kısım, öğrenciye (size) egzersiz olarak bırakılmıştır.
  
  <details>
  <summary>Çözüm içeride</summary>
   --------------------------------------------------------------------------------------------------
    
  Resmî düzeltme:
  - https://github.com/GNOME/libxml2/commit/932cc9896ab41475d4aa429c27d9afd175959d74
    
   </details> 

Alternatif olarak, LibXML'in daha yeni bir sürümünü indirebilir ve hatanın düzeltildiğini kontrol edebilirsiniz.
  
</details>
