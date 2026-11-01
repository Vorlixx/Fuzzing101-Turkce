# Fuzzing-101
> **Not:** Bu depo, [Antonio Morales](https://github.com/antonio-morales)'in [**Fuzzing101**](https://github.com/antonio-morales/Fuzzing101) kursunun resmi olmayan **Türkçe çevirisidir**. Orijinal içerik [Apache-2.0](./LICENSE) lisansıyla sunulmuştur; tüm haklar orijinal yazarına aittir. Ayrıntılar için [NOTICE.md](./NOTICE.md) dosyasına bakabilirsiniz. Çeviri hatalarını [Issues](https://github.com/Vorlixx/Fuzzing101-Turkce/issues) üzerinden bildirebilirsiniz.

Gerçek bir uzman gibi fuzzing öğrenmek istiyorsunuz ama başlamayı bilmiyor musunuz?

Öyleyse, bu kurs tam size göre!

**10 gerçek hedef, 10 egzersiz.** Hepsini, yani 10'unu da çözebilecek misiniz?

## Yapı

| Egzersiz No.  | Hedef | Bulunacak CVE'ler | Tahmini süre | Ana konular |
| ------------- | ------------- | ------------- |  ------------- | ------------- |
| [Egzersiz 1](Exercise%201) | Xpdf  | CVE-2019-13288 | 120 dakika | Afl-clang-fast, Afl-fuzz, GDB |
| [Egzersiz 2](Exercise%202)  | libexif  |  CVE-2009-3895, CVE-2012-2836 | 6 saat | Afl-clang-lto, Kütüphaneleri fuzz'lamak, Eclipse IDE|
| [Egzersiz 3](Exercise%203)  | TCPdump  | CVE-2017-13028 | 4 saat | ASan, Sanitizer'lar |
| [Egzersiz 4](Exercise%204)  | LibTIFF  | CVE-2016-9297 | 3 saat | Kod kapsamı (code coverage), LCOV |
| [Egzersiz 5](Exercise%205)  | Libxml2  | CVE-2017-9048 | 3 saat | Sözlükler (dictionaries), Temel paralelleştirme, Komut satırı argümanlarını fuzz'lamak  |
| [Egzersiz 6](Exercise%206)  | GIMP  | CVE-2016-4994, Bonus hatalar | 7 saat | Persistent (kalıcı) fuzzing, Etkileşimli uygulamaları fuzz'lamak|
| [Egzersiz 7](Exercise%207)  | VLC media player | CVE-2019-14776 | 6 saat | Kısmi enstrümantasyon (partial instrumentation), Fuzzing harness |
| [Egzersiz 8](Exercise%208)  | Adobe Reader  | | 8 saat | Kapalı kaynaklı uygulamaları fuzz'lamak, QEMU enstrümantasyonu|
| [Egzersiz 9](Exercise%209)  | 7-Zip  | CVE-2016-2334 | 8 saat | WinAFL, Windows Uygulamalarını fuzz'lamak|
| [**Egzersiz 10 (Final Meydan Okuma)**](Exercise%2010)  | **Google Chrome / V8**  | **CVE-2019-5847** | **8 saat** | **Fuzzilli, JavaScript motorlarını fuzz'lamak** |

## Değişiklik Günlüğü

- 14.02.2022: Egzersiz 5'teki bazı 'wget' yazım hataları düzeltildi
- 25.11.2021: Egzersiz 3 bazı düzeltmelerle güncellendi.


## Bu kurs kimler için?

- Fuzzing temellerini öğrenmek isteyen herkes
- Gerçek yazılım projelerinde güvenlik açıklarını nasıl bulacaklarını öğrenmek isteyen herkes.

## Gereksinimler

- Bu kurs için ihtiyacınız olan tek şey, internet bağlantısına sahip, çalışan bir Linux sistemidir. Egzersizlerde uygun bir VMware imajı bulacaksınız.
- En azından temel düzeyde Linux bilgisi şiddetle tavsiye edilir.
- Tüm egzersizler Ubuntu **20.04.2 LTS** üzerinde test edilmiştir. Buradan indirebilirsiniz: [buradan](https://ubuntu.com/download/desktop/thank-you?version=20.04.2.0&architecture=amd64)
- Bu kursta fuzzing egzersizlerini çözmek için, Michał "lcamtuf" Zalewski'nin AFL'sinin daha yeni ve daha üstün bir fork'u olan [AFL++](https://github.com/AFLplusplus/AFLplusplus) aracını kullanacağız.

## Fuzzing nedir?

**Fuzz testi (veya fuzzing)**, programa rastgele/mutasyona uğramış (mutated) girdi değerleri verilerek programın istisnalar/çökmeler (exceptions/crashes) açısından izlenmesine dayalı otomatik bir yazılım test tekniğidir.

[AFL](https://github.com/google/AFL), [libFuzzer](https://llvm.org/docs/LibFuzzer.html) ve [HonggFuzz](https://github.com/google/honggfuzz), gerçek dünya uygulamaları söz konusu olduğunda en başarılı üç fuzzer'dır. Üçü de **Coverage-guided (kapsam güdümlü) evolutionary (evrimsel)** fuzzer'lara örnektir.

###  Coverage-guided evolutionary fuzzer

- **Evolutionary (Evrimsel)**: Evrimsel algoritmalardan esinlenen metaheuristic bir yaklaşımdır; temel olarak başlangıç alt kümesinin (seed'ler) zaman içinde, bir seçim kriteri (örn. kapsam/coverage) kullanılarak evrimleşmesinden ve mutasyona uğramasından oluşur.

- **Coverage-guided (Kapsam güdümlü)**: Yeni çökmeler bulma şansını artırmak için, kapsam güdümlü fuzzer'lar farklı girdiler arasında kod kapsamı (code coverage) verilerini toplar ve karşılaştırır (genellikle enstrümantasyon yoluyla) ve yeni yürütme yollarına (execution paths) götüren girdileri seçer.


<img src="./Diagram.png">

<p align="center">
  Bir coverage-guided evolutionary fuzzer'ın kapsam (coverage) toplama sürecinin basitleştirilmiş gösterimi
</p>

## Teşekkürler

Yardımlarından dolayı teşekkürler:
- [Xavier RENE-CORAIL](https://github.com/xcorail)
- [Alan Vivona](https://github.com/alanvivona)
- [Jason White](https://github.com/misfir3)
- [Octavio Gianatiempo](https://github.com/ogianatiempo)
- [van Hauser](https://github.com/vanhauser-thc)
- [Marc Poulhiès](https://github.com/dkm)
- [Xu Hanyu](https://github.com/Mundi-Xu)
- [tclan126](https://github.com/tclan126)
- [epi052](https://github.com/epi052)
- [Jeremias Gomes](https://github.com/j3r3mias)
