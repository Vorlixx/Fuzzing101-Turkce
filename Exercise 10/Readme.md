# Egzersiz 10 (Final Meydan Okuma) - V8 motoru

Bu egzersizde **V8**, Google'ın JavaScript ve WebAssembly motorunu fuzz'layacağız. Amaç, v8 7.5 sürümünde [**CVE-2019-5847**](https://nvd.nist.gov/vuln/detail/CVE-2019-5847) için bir çökme/PoC bulmaktır.

<details>
  <summary>CVE-2019-5847 güvenlik açığı hakkında daha fazla bilgi için, buraya tıklayın!</summary>
  --------------------------------------------------------------------------------------------------------
  
  **CVE-2019-5847**, özel olarak hazırlanmış (crafted) bir dosya yoluyla sonsuz özyinelemeye (infinite recursion) yol açabilen bir güvenlik açığıdır.
  
  Heap tabanlı bellek bozulması (heap-based memory corruption), heap veri alanında meydana gelen bir bellek bozulması türüdür ve genellikle açık dinamik bellek yönetimiyle (malloc() ve free() fonksiyonlarıyla bellek ayırma/ayırma iptali) ilgilidir.

Sonuç olarak, uzaktan bir saldırgan, bu sorunu etkilenen kütüphaneyi kullanan bir uygulamanın bağlamında (context) keyfi kod çalıştırmak için kullanabilir.

Heap tabanlı bellek bozulması (Heap-based memory corruption) güvenlik açıkları hakkında daha fazla bilgiyi şu bağlantıda bulabilirsiniz: https://cwe.mitre.org/data/definitions/122.html
  
</details>

## Neler öğreneceksiniz?
Bu egzersizi tamamladığınızda şunları bilmiş olacaksınız: 
- JavaScript motorlarını fuzz'lamak için Fuzzilli'yi nasıl kullanacağınızı

## Başlamadan Önce Okuyun
- Egzersizi, çözüme bakmadan **kendiniz çözmenizi** öneririm. Gücünüzün yettiği kadar çabalayın ve ancak takılırsanız aşağıdaki örnek çözüme bakın.
- Yeni bir güvenlik açığı bulursanız, **lütfen projeye bir güvenlik raporu gönderin**. Bu süreç hakkında yardıma ihtiyacınız olursa ya da herhangi bir şüpheniz olursa [GitHub Security Lab](mailto:securitylab.github.com) size yardımcı olabilir :)

## İletişim
Takıldınız mı ve yardım mı arıyorsunuz? Bu kursu daha iyi hale getirmek için önerileriniz mi var, yoksa daha fazla benzer içerik oluşturmamız için sadece olumlu geri bildirimler mi? 
Fuzzing deneyiminizi toplulukla paylaşmak ister misiniz?
GitHub Security Lab Slack'ine katılın ve `#fuzzing` kanalına gidin. [GitHub Security Lab Slack daveti isteyin](mailto:securitylab-social@github.com?subject=Request%20an%20invite%20to%20the%20GitHub%20Security%20Lab%20Slack)

## Ortam

Tüm egzersizler **Ubuntu 18.04 LTS** üzerinde test edilmiştir. Rehberi takip edebilmek için **aynı işletim sistemi sürümünü** kullanmanızı şiddetle tavsiye ederim. Ubuntu 18.04 LTS VMware imajını [burada](https://drive.google.com/file/d/1925mpjbIL8ugc7WcOooIatYWnMKSX-g4/view?usp=sharing) bulabilirsiniz. VMware yerine VirtualBox da kullanabilirsiniz.

Bu sanal makinenin kullanıcı adı / şifresi `fuzz` / `fuzz`'tur.

## Fuzzilli

Fuzzilli, JavaScript'e dönüştürülebilen ve mutasyona uğratılabilen özel bir ara dil ("FuzzIL") temel alan, dinamik dil yorumlayıcıları için (coverage-)güdümlü bir fuzzer'dır. Samuel Groß (saelo@google.com) tarafından yazılmakta ve bakımı yapılmaktadır.

Fuzzilli hakkında daha fazla bilgiyi [resmî repoda](https://github.com/googleprojectzero/fuzzilli/) bulabilirsiniz

### Bağımlılıkları kurun

Ortamınızı tamamen hazır hale getirmek için bazı ek araçlar kurmanız gerekebilir (örn. Swift, Git)

Şu bağımlılıkları kurarak başlayalım:
```
sudo apt --yes install clang libcurl3 libpython2.7 libpython2.7-dev libcurl4 git
```

Şimdi, Swift'i indirebiliriz:
```
cd $HOME
wget https://swift.org/builds/swift-4.2.1-release/ubuntu1804/swift-4.2.1-RELEASE/swift-4.2.1-RELEASE-ubuntu18.04.tar.gz
tar xzvf swift-4.2.1-RELEASE-ubuntu18.04.tar.gz
sudo mv swift-4.2.1-RELEASE-ubuntu18.04 /usr/share/swift
```

Ayrıca PATH ortam değişkenini (envvar) yapılandırmamız gerekiyor:
```
echo "export PATH=/usr/share/swift/usr/bin:$PATH" >> ~/.bashrc
source  ~/.bashrc
```

### Fuzzilli'yi kurun

Fuzzilli'yi indirme ve derleme zamanı:
```
cd $HOME
wget https://github.com/googleprojectzero/fuzzilli/archive/refs/tags/v0.9.zip
unzip v0.9.zip
cd fuzzilli-0.9/
swift build -c release -Xlinker='-lrt'
```

## V8 motorunu indirin ve derleyin

### depot_tools'u indirin ve kurun

V8, checkout'ları ve kod incelemelerini (code reviews) yönetmek için depot_tools adlı bir betik paketi kullanır. depot_tools paketi; gclient, gcl, git-cl, repo ve diğerlerini içerir. Şununla kurabiliriz:
```
cd $HOME
mkdir depot_tools && cd depot_tools
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
echo "export PATH=`pwd`/depot_tools:$PATH" >> ~/.bashrc
source  ~/.bashrc
```

### V8 kaynak kodunu edinin:
```
cd $HOME
mkdir Fuzzing_v8_75 && cd Fuzzing_v8_75
fetch v8
cd v8
git checkout 1ca088652d3aad04caceb648bcffef100bc4abc0
gclient sync
```

### V8'i derleyin ve test edin:

Her şeyden önce, derleme bağımlılıklarını kurmamız gerekir:
```
./build/install-build-deps.sh
```

gn kullanarak derleme dosyalarını (build files) oluşturun:
```
gn gen out/Release "--args=is_debug=false"
```

Bundan sonra, şu komut satırıyla derleyebiliriz (biraz sürebilir):
```
ninja -C out/Release
```

Şimdi, d8 binary'sini şu komutla kontrol edebiliriz:
```
./out/Release/d8 ./test/fuzzer/parser/hello-world
```

ve şuna benzer bir şey görmelisiniz:

![](Images/1.png)

## Fuzzing zamanı

v8'i coverage enstrümantasyonuyla derleyin:
```
cd $HOME/Fuzzing_v8_75/v8
cp ../../fuzzilli-0.9/Targets/V8/v8.patch ./
gn gen out/fuzzbuild --args='is_debug=false dcheck_always_on=true v8_static_library=true v8_enable_slow_dchecks=true v8_enable_v8_checks=true v8_enable_verify_heap=true v8_enable_verify_csa=true v8_enable_verify_predictable=true sanitizer_coverage_flags="trace-pc-guard" target_cpu="x64"'
ninja -C ./out/fuzzbuild
```

Fuzzilli klasörüne geçin:
```
cd $HOME/fuzzilli-0.9
```

Her şeyden önce, core dump dosyalarının oluşturulmasını devre dışı bırakmamız gerekiyor:
```
sudo sysctl -w 'kernel.core_pattern=|/bin/false'
```

Şimdi fuzzilli'yi şu komutla çalıştırabiliriz:
```
swift run -Xlinker='-lrt' -c release FuzzilliCli --profile=v8 '/home/fuzz/Fuzzing_v8_75/v8/out/fuzzbuild/d8'
```

Seçeneklerin tam listesi için şunu yazın:
```
swift run -Xlinker='-lrt' FuzzilliCli --help
```

Her şey yolunda gittiyse, şuna benzer bir şey görmelisiniz:

![](Images/2.png)

## Kendin yap!
Bu meydan okumayı çözmek için şunları yapmanız gerekir:

1) Güvenlik açığı bulunan sürümü indirmek ve derlemek
2) Güvenlik açığı bulunan kodu hedeflemek için özel bir mutatör (custom mutator) tanımlamak
3) Birkaç benzersiz çökme elde edene kadar Fuzzilli ile fuzz'lamak
4) Güvenlik açığı için PoC bulmak amacıyla çökmeleri incelemek (triage)
5) PoC'nuzu göndermek (aşağıya bakın)

## Meydan okuma kuralları
Kurallar şunlardır:
- Lütfen çözümünüzü açıklamayın (paylaşmayın)
- Kazanan olmak için geçerli bir çökme/PoC dosyası sağlamanız gerekir
- Toplamda 2 kazanan olacak
- Her hafta yeni bir ipucu yayınlayacağım
- Çözümünüzü **28 Şubat** tarihine kadar antoniomoralesmaldonado@gmail.com adresine gönderin
- Kazananlar, [GitHub Shop](https://thegithubshop.com/)'ta harcayacakları bir kupon alacaklardır

## İpuçları
- 1. ipucu: 

```
git checkout 7.5.288.22
```

## Bu meydan okuma 1 Mart 2022'de sona erdi. Gönderimleriniz için teşekkürler!! :smiley:
