# Egzersiz 9 - 7-Zip

Bu egzersizde **7-Zip** dosya arşivleyicisini (file archiver) fuzz'layacağız. Amaç, 7-Zip 15.05 sürümünde [**CVE-2016-2334**](https://nvd.nist.gov/vuln/detail/CVE-2016-2334) için bir çökme/PoC bulmaktır.

--> Böyle harika bir buluş için [icewall](https://github.com/icewall)'a tebrikler.

<details>
  <summary>CVE-2016-2334 güvenlik açığı hakkında daha fazla bilgi için, buraya tıklayın!</summary>
  --------------------------------------------------------------------------------------------------------
  
  **CVE-2016-2334**, özel olarak hazırlanmış (crafted) bir HFS+ imajı yoluyla tetiklenebilen heap tabanlı bir tampon taşması (heap-based buffer overflow) durur.
  
Heap tabanlı tampon taşması, heap veri alanında meydana gelen bir tampon taşması türüdür ve genellikle açık dinamik bellek yönetimiyle (malloc() ve free() fonksiyonlarıyla bellek ayırma/ayırma iptali) ilgilidir.

Sonuç olarak, uzaktan bir saldırgan, bu sorunu etkilenen kütüphaneyi kullanan bir uygulamanın bağlamında (context) keyfi kod çalıştırmak için kullanabilir.

Heap tabanlı tampon taşması (Heap-based buffer overflow) güvenlik açıkları hakkında daha fazla bilgiyi şu bağlantıda bulabilirsiniz: https://cwe.mitre.org/data/definitions/122.html
  
</details>


## Neler öğreneceksiniz?
Bu egzersizi tamamladığınızda şunları bilmiş olacaksınız:
- Windows uygulamalarını fuzz'lamak için WinAFL'nin nasıl kullanılacağını

## Başlamadan Önce Okuyun
- Egzersizi, çözüme bakmadan **kendiniz çözmenizi** öneririm. Gücünüzün yettiği kadar çabalayın ve ancak takılırsanız aşağıdaki örnek çözüme bakın.
- AFL, deterministik olmayan (non-deterministic) bir test algoritması kullanır, bu yüzden iki fuzzing oturumu asla aynı olmaz. Bu yüzden **sabit bir tohum değeri (seed) ayarlamanızı (-s 123)** şiddetle tavsiye ederim. Bu sayede fuzzing sonuçlarınız burada gösterilenlere benzer olacak ve egzersizleri daha kolay takip etmenizi sağlayacak.  
- Yeni bir güvenlik açığı bulursanız, **lütfen projeye bir güvenlik raporu gönderin**. Bu süreç hakkında yardıma ihtiyacınız olursa ya da herhangi bir şüpheniz olursa [GitHub Security Lab](mailto:securitylab.github.com) size yardımcı olabilir :)

## İletişim
Takıldınız mı ve yardım mı arıyorsunuz? Bu kursu daha iyi hale getirmek için önerileriniz mi var, yoksa daha fazla fuzzing içeriği oluşturmamız için sadece olumlu geri bildirimler mi? 
Fuzzing deneyiminizi toplulukla paylaşmak ister misiniz?
GitHub Security Lab Slack'ine katılın ve `#fuzzing` kanalına gidin. [GitHub Security Lab Slack daveti isteyin](mailto:securitylab-social@github.com?subject=Request%20an%20invite%20to%20the%20GitHub%20Security%20Lab%20Slack)

## Ortam

Bu egzersiz **Windows 10** üzerinde test edilmiştir. Farklı fuzzing sonuçlarını önlemek için **aynı işletim sistemi sürümünü** kullanmanızı şiddetle tavsiye ederim

Windows 10 ücretsiz VMware imajını şu [bağlantıdan](https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/) indirebilirsiniz. VMware yerine VirtualBox da kullanabilirsiniz.

![](Images/Image0.png)

Sanal makinenin şifresi **"Passw0rd!"**'dur

## Kendin yap!
Bu egzersizi tamamlamak için şunları yapmanız gerekir:
1) WinAFL'yi (ve gerekli bağımlılıkları) indirmek ve derlemek
2) HFS+ örneklerinden oluşan bir seed corpus (tohum külliyatı) oluşturmak
3) İsteğe bağlı: HFS+ dosya formatı için bir fuzzing sözlüğü (dictionary) oluşturmak
4) Bir çökme elde edene kadar 7-Zip'i fuzz'lamak
5) Güvenlik açığı için PoC bulmak amacıyla çökme incelemesi (crash triage) yapmak
6) İsteğe bağlı: Fuzzing sürecini hızlandırmak için daha iyi bir *target_offset* bulmak

**Tahmini süre = 8 saat**

---------------------------------------------------------------------------------------------------------------------------------------------------

<details>
  <summary>SPOİLER UYARISI! : Çözüm içeride</summary>
 
### Önceki adımlar  
  
Her şeyden önce, sistemimizde **Visual Studio derleyicisinin** kurulu olması gerekiyor. Bu egzersiz için Visual Studio 2019 kullanmanızı öneririm. Visual Studio 2019 Community Edition kurucusunu [burada](./Resources/vs_community.exe) bulabilirsiniz.
  
Ardından **"Desktop development with C++"** paketini seçip kurmamız gerekir:
  
![](Images/Image1.png)
  
Ayrıca **DynamoRIO 8.0.0**'a da ihtiyacımız olacak. DynamoRIO Windows binary paketini [buradan](https://github.com/DynamoRIO/dynamorio/releases/download/release_8.0.0-1/DynamoRIO-Windows-8.0.0-1.zip) edinebiliriz.
  
Ardından, zip içeriğini şöyle masaüstüne (Desktop) çıkarmamız gerekir:

![](Images/Image2.png)

### WinAFL'yi indirin ve derleyin
  
Şimdi, WinAFL'yi resmî repodan indirebiliriz: https://github.com/googleprojectzero/winafl
  
Bundan sonra, **"Developer Command Prompt for VS2019"** uygulamasını açın ve çalışma dizinini WinAFL dizinine değiştirin. Ardından şunu yazın:
  
```
mkdir build32
cd build32
cmake -G"Visual Studio 16 2019" -A Win32 .. -DDynamoRIO_DIR=C:\Users\IEUser\Desktop\DynamoRIO-Windows-8.0.0-1\cmake
cmake --build . --config Release
```
  
burada `C:\Users\IEUser\Desktop\DynamoRIO-Windows-8.0.0-1\cmake`, kendi sisteminizdeki DynamoRIO yoludur.
  
Her şey yolunda gittiyse, artık tüm WinAFL binary'lerini `winafl-master\build32\bin\Release` klasöründe bulabilirsiniz:
  
![](Images/Image3.png)
  
**Dikkatli olun! Bu klasörü "bin32" klasörüyle karıştırmayın**
  
### 7-Zip'i indirin
  
Şimdi, 7-Zip 15.05'i kurma zamanı. Kurucuyu [burada](./Resources/7z1505.exe) bulabilirsiniz.

### Seed corpus (tohum külliyatı) oluşturma
  
Seed külliyatınızı beslemek için bazı HFS+ imajları oluşturmanızı öneririm. Bu, Mac OS üzerinde önemsiz bir iştir.
  
Linux'ta **hfsprogs** yardımcı programını kullanabilirsiniz. Windows'ta **Paragon HFS+** (ücretsiz denemeli ticari yazılım) kullanabilirsiniz.
 
İşi kolaylaştırmak için bir HFS örnek dosyasını [burada](./Resources/example.img) bulabilirsiniz.
  
**Dikkat! Bu sadece bir başlangıç noktası, kendi başınıza biraz ekstra iş yapmanız gerekecek**
  
### Fuzzing zamanı  
  
WinAFL komut satırı, AFL++'dan biraz farklıdır. Bu yeni seçeneklerin kısa açıklamasına bakalım:
- *-coverage_module* : Kapsam (coverage) kaydı tutulacak modül. Birden fazla modül bayrağı desteklenir
- *-target_module* : Fuzz'lanacak hedef fonksiyonu içeren modül
- *-target_offset* : Fuzz'lanacak metodun modül başından itibaren offset'i

[Egzersiz 8'de](../../main/Exercise%208) yaptığımız gibi, fuzzer'ın döngüde döneceği uygun bir fonksiyon offset'i bulmamız gerekir: 
  
![](Images/Image4.png)
  
Fonksiyonun modül başından itibaren offset'ine ihtiyacımız var. Temel adres (base address) `0x400000` olduğu için, `0x42F3B3 - 0x400000` yapacağız ve **target_offset** argümanı olarak `0x02F3B3` elde edeceğiz.
  
Şimdi, hedefin DynamoRIO altında doğru çalıştığını kontrol edelim:
```
C:\Users\IEUser\Desktop\DynamoRIO-Windows-8.0.0-1\bin32\drrun.exe -c winafl.dll -debug -target_module 7z.exe -target_offset 0x02F3B3 -fuzz_iterations 10 -nargs 2 -- "C:\Program Files (x86)\7-Zip\7z.exe" l C:\Users\IEUser\Desktop\input\test.img
```
  
Hedef fonksiyonunuza ait 10 kez çalıştırılmaya karşılık gelen çıktıyı görmelisiniz, sonrasında hedef çalıştırılabilir dosya (executable) çıkacaktır.
  
Son olarak, fuzzer'ı şu komutla çalıştırabiliriz:
```
afl-fuzz.exe -i C:\Users\IEUser\Desktop\afl_in -o C:\Users\IEUser\Desktop\afl_out -t 2000 -D C:\Users\IEUser\Desktop\DynamoRIO-Windows-8.0.0-1\bin32 -- -coverage_module 7z.exe -coverage_module 7z.dll -target_module 7z.exe -target_offset 0x02F3B3 -nargs 2 -- "C:\Program Files (x86)\7-Zip\7z.exe" e -y @@  
```

Ve WinAFL'nin çalıştığını görmelisiniz:
  
![](Images/Image5.png)

  
### Daha iyi bir target_offset bulmak
  
Egzersizin son adımı, fuzzing sürecini hızlandırmak için daha iyi bir target_offset bulmaktır. Bu son kısım, öğrenciye (size) egzersiz olarak bırakılmıştır.
  
**İpucu**: 7-Zip açık kaynaklıdır. Yani onu debug modunda derleyebilir ve hata ayıklayıcınızda fonksiyon adlarını görebilirsiniz ;)
 
