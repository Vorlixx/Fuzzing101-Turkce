# Egzersiz 8 - Adobe Reader

Bu egzersizde **Adobe Reader** uygulamasını fuzz'layacağız. Amaç, Adobe Reader 9.5.1 sürümünde bir Sınır Dışı (Out-of-bounds) güvenlik açığı bulmaktır.

## Neler öğreneceksiniz?
Bu egzersizi tamamladığınızda şunları bilmiş olacaksınız:
- Kapalı kaynaklı (closed-source) uygulamaları fuzz'lamak için AFL++'ın QEMU modunu nasıl kullanacağınızı
- QEMU modunda persistent (kalıcı) modun nasıl etkinleştirileceğini
- Binary-only bir sanitizer olan QASAN'ı nasıl kullanacağınızı

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
1) PDF örneklerinden oluşan bir seed corpus (tohum külliyatı) oluşturmak
2) Persistent (kalıcı) modu etkinleştirmek
3) Adobe Reader'ı, bazı çökmeler elde edene kadar QEMU modunda fuzz'lamak
4) Güvenlik açığı için PoC bulmak amacıyla çökmeleri incelemek (triage)


**Tahmini süre = 8 saat**

---------------------------------------------------------------------------------------------------------------------------------------------------

<details>
  <summary>SPOİLER UYARISI! : Çözüm içeride</summary>

### AFL++'ın QEMU kurulumu

Her şeyden önce, afl-qemu'nun sistemimizde kurulu olması gerekiyor. Şu komutla kontrol edebilirsiniz:

```
afl-qemu-trace --help
```

ve şuna benzer bir şey görmelisiniz:

![](Images/0.png)

Henüz kurulu değilse, şununla derleyip kurabilirsiniz:

```
sudo apt install ninja-build libc6-dev-i386
cd ~/Downloads/AFLplusplus/qemu_mode/
CPU_TARGET=i386 ./build_qemu_support.sh
make distrib
sudo make install
```

burada ``/Downloads/AFLplusplus/``, [AFL++ kök klasörüdür](../Exercise%201#install-afl)

### Adobe Reader kurulumu

Bağımlılıkları kurun:
```
sudo apt-get install libxml2:i386
```

``AdbeRdr9.5.1-1_i386linux_enu.deb`` dosyasını indirin ve sıkıştırmasını açın:

```
wget ftp://ftp.adobe.com/pub/adobe/reader/unix/9.x/9.5.1/enu/AdbeRdr9.5.1-1_i386linux_enu.deb
```

Şimdi, şu komutla kurabiliriz:

```
sudo dpkg -i AdbeRdr9.5.1-1_i386linux_enu.deb
```

Ardından sadece şunu yazın:

```
./opt/Adobe/Reader9/bin/acroread
```

Lisans Sözleşmesi'ni kabul edin ve Adobe Reader arayüzünü görmelisiniz:

![](Images/4.png)

Eğer şunu yazarsanız

```
/opt/Adobe/Reader9/bin/acroread -help 
```

kullanılabilir komut satırı seçeneklerinin listesini de görebilirsiniz

### Seed corpus (tohum külliyatı) oluşturma

Bu egzersiz için, **SafeDocs "Issue Tracker" Corpus**'tan indirilmiş bir külliyat kullanacağım. Bu PDF külliyatı hakkında daha fazla bilgiyi [burada](https://www.pdfa.org/a-new-stressful-pdf-corpus/) bulabilirsiniz.

Külliyatı indirin ve sıkıştırmasını açın:
```
wget https://corpora.tika.apache.org/base/packaged/pdfs/archive/pdfs_202002/libre_office.zip
unzip libre_office.zip -d extracted
```

Şimdi, fuzzing sürecini hızlandırmak için yalnızca **2 kB'den küçük dosyaları** kopyalayacağız:
```
mkdir -p $HOME/fuzzing_adobe/afl_in
find ./extracted -type f -size -2k \
    -exec cp {} $HOME/fuzzing_adobe/afl_in \;
```


### İlk yaklaşım

Kapalı kaynaklı uygulamaları fuzz'lamanın en basit yolu, sadece afl-fuzz'ı **-Q argümanıyla** çalıştırmaktır.

afl-fuzz'ı çalıştırırken dikkatli olmanız gerekir, çünkü ``/opt/Adobe/Reader9/bin/acroread`` bir shell script'tir. Gerçek binary yolu şudur: ``/opt/Adobe/Reader9/Reader/intellinux/bin/acroread``.

Ama çalıştırmayı denerseniz şöyle bir hata alırsınız: ``acroread must be executed from the startup script``. Bu yüzden ``ACRO_INSTALL_DIR`` ve ``ACRO_CONFIG`` ortam değişkenlerini (envvar) ayarlamamız gerekiyor. Ayrıca, dinamik bağlanabilir kütüphanelerin (dynamically linkable libraries) aranacağı dizini tanımlamak için ``LD_LIBRARY_PATH``'i de ayarlayacağız.

Bunu söyledikten sonra, fuzzer'ı şu komutla çalıştırabilirsiniz:
```
ACRO_INSTALL_DIR=/opt/Adobe/Reader9/Reader ACRO_CONFIG=intellinux LD_LIBRARY_PATH=$LD_LIBRARY_PATH:'/opt/Adobe/Reader9/Reader/intellinux/lib' afl-fuzz -Q -i ./afl_in/ -o ./afl_out/ -t 2000 -- /opt/Adobe/Reader9/Reader/intellinux/bin/acroread -toPostScript @@
```

Ve AFL++'ın çalıştığını görmelisiniz:

![](Images/5.png)

Şu anda ise, fuzzing hızı gerçekten yavaş: makinemde yaklaşık 7 exec/s. Peki, fuzzing hızını nasıl artırabiliriz?

Cevap... Persistent (kalıcı) fuzzing kullanmak!

### Persistent yaklaşım

[Egzersiz 6'da](../Exercise%206) gördüğümüz gibi, ``AFL_LOOP`` eklemek, AFL++'a persistent modu etkinleştirmek istediğimizi söylememizin yoludur. Ancak bu durumda, kaynak koda erişimimiz yok.

Bunun yerine, persistent döngünün başlangıcını belirtmek için AFL_QEMU_PERSISTENT_ADDR kullanabiliriz. Bu adresi bir fonksiyonun başına ayarlamanızı öneririm. AFL_QEMU persistent seçenekleri hakkında daha fazla bilgiyi [burada](https://github.com/AFLplusplus/AFLplusplus/blob/stable/qemu_mode/README.persistent.md) bulabilirsiniz.

Uygun bir offset bulmak için IDA veya Ghidra gibi bir disassembler kullanabiliriz. Benim durumumda, şu offset'i seçtim ``0x08546a00``:

![](Images/6.png)

Disassembler kullanmaya alternatif daha kolay bir yol var: **callgrind** kullanmak.

Her şeyden önce, valgrind ve kcachegrind'i şu komut satırıyla kuracağız:

```
sudo apt-get install valgrind
sudo apt-get install kcachegrind
```

Şimdi, bir callgrind raporu şununla oluşturabiliriz:

```
ACRO_INSTALL_DIR=/opt/Adobe/Reader9/Reader ACRO_CONFIG=intellinux LD_LIBRARY_PATH=$LD_LIBRARY_PATH:'/opt/Adobe/Reader9/Reader/intellinux/lib' valgrind --tool=callgrind /opt/Adobe/Reader9/Reader/intellinux/bin/acroread -toPostScript [samplePDF]
```

burada ``samplePDF``, bir PDF örneğinin yoludur. Bu, mevcut klasörünüzde bir ``callgrind.out`` çıktı dosyası oluşturacaktır. Şimdi bu raporu **kachegrind** çalıştırarak görselleştirebilirsiniz. Sadece şunu yazın:

```
kcachegrind 
```

ve şuna benzer bir şey görmelisiniz:

![](Images/8.png)

Sadece 1 kez çalıştırılan fonksiyonları belirlemek için kcachegrind'deki ``count`` alanına bakmanızı ve afl-fuzz'da **%90'ın üzerinde bir kararlılık (stability) puanı** elde etmeye çalışmanızı öneririm.
  
Ayrıca, genel amaçlı yazmaçların (general purpose registers) orijinal değerini kaydeden ve her persistent döngüsünde bunları geri yükleyen **AFL_QEMU_PERSISTENT_GPR=1** ortam değişkenini de ayarlayacağız.

Şimdi, fuzzer'ı şu komut satırıyla çalıştırabiliriz

```
AFL_QEMU_PERSISTENT_ADDR=0x085478AC AFL_QEMU_PERSISTENT_GPR=1 ACRO_INSTALL_DIR=/opt/Adobe/Reader9/Reader ACRO_CONFIG=intellinux LD_LIBRARY_PATH=$LD_LIBRARY_PATH:'/opt/Adobe/Reader9/Reader/intellinux/lib' afl-fuzz -Q -i ./afl_in/ -o ./afl_out/ -t 2000 -- /opt/Adobe/Reader9/Reader/intellinux/bin/acroread -toPostScript @@
```
Gördüğünüz gibi, yürütme süresinde x4'lük bir iyileşme sağlıyor. Fena değil!

![](Images/7.png)

Küçük projeler için bu yaklaşım yeterli olabilir. Gelecek bir egzersizde daha hızlı bir yaklaşımı açıklayacağım: **özel bir harness yazmak**.

### İnceleme (Triage)

Bu son adımda, OOB read güvenlik açığımız için bir PoC bulmaya çalışacağız.

Ne yazık ki, afl-qemu-trace'e çökme dosyasını verirsek:

```
ACRO_INSTALL_DIR=/opt/Adobe/Reader9/Reader ACRO_CONFIG=intellinux LD_LIBRARY_PATH=$LD_LIBRARY_PATH:'/opt/Adobe/Reader9/Reader/intellinux/lib' /usr/local/bin/afl-qemu-trace -- /opt/Adobe/Reader9/Reader/intellinux/bin/acroread -toPostScript [crashFilePath] 
```

(burada ``crashFilePath``, çökme dosyasının yoludur), görebileceğimiz tek şey şöyle bir mesajdır

![](Images/10.png)

**QASan** kullanarak daha detaylı bir stacktrace elde edebiliriz. Onu etkinleştirmek için sadece ``AFL_USE_QASAN=1`` ayarlamamız gerekir. QASAN hakkında daha fazla bilgiyi [burada](https://github.com/andreafioraldi/qasan) bulabilirsiniz. Şimdi şunu yazıyoruz:

```
AFL_USE_QASAN=1 ACRO_INSTALL_DIR=/opt/Adobe/Reader9/Reader ACRO_CONFIG=intellinux LD_LIBRARY_PATH=$LD_LIBRARY_PATH:'/opt/Adobe/Reader9/Reader/intellinux/lib' /usr/local/bin/afl-qemu-trace -- /opt/Adobe/Reader9/Reader/intellinux/bin/acroread -toPostScript [crashFilePath] 
```

ve daha güzel bir stacktrace alacağız:

![](Images/11.png)

</details>
