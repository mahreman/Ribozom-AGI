# Ribozom 🧬 
**A Dependency-Free, GPU-Free, Self-Correcting NLP Engine in Pure C**

Ribozom, günümüzün "kara kutu" (black-box) Büyük Dil Modellerine (LLM) alternatif olarak sıfırdan, hiçbir harici kütüphane kullanılmadan yazılmış biyolojik esinli bir doğal dil işleme motorudur. Dünyayı devasa bir parametre çorbasına çevirmek yerine, dilin fraktal ve sentaktik yapısını O(L) karmaşıklığıyla **LCRS Trie** mimarisine oturtur.

> **Dürüst Durum Özeti:** > *Ribozom canlı etkileşimden öğrenip kalıcı hafıza güncelleyebilen, GPU'suz, saf C bir sistemdir. Bu, modern LLM'lerin (In-Context Learning dışında) yapamadığı kalıcı "Online Learning"in minimal ve şeffaf bir kanıtıdır.*

## 🔬 Temel Özellikler

* **Sıfır Bağımlılık (Zero-Dependency):** Sadece standart C kütüphaneleri. Framework yok, Python yok, CUDA yok. C derleyicisinin olduğu her yerde, bir hesap makinesinde bile çalışır.
* **Canlı Düzeltme Döngüsü (Live Correction Loop):** Model hata yaptığında `"hayır, doğrusu X"` diyerek çıkarım (inference) anında sistemi eğitebilirsiniz. Ribozom, ağırlıklarını anında günceller ve bunu `ribo.bin` dosyasına kalıcı olarak yazar.
* **Lizozom Mimarisi (RAG in C):** Yeni öğrenilen "gerçekler" (facts), büyük veri setinin yerçekimine yenilmemesi için biyolojik bir sindirim kesesi olan *Lizozom* modülünde Jaccard benzerliği ile doğrudan eşleşerek (Bag-of-words) bypass edilir.
* **Otonom Morfoloji Keşfi:** Türkçe gibi eklemeli (agglutinative) dillerin yapısını, hiçbir dilbilgisi kuralı verilmeden, sadece istatistiksel dağılım frekanslarıyla kendi kendine çözer (`kedi` ↔ `kedici`, vb.).
* **Dinamik Sinyal Normalizasyonu (SNR):** Sistem, kendi tahmin özgüvenini (Signal-to-Noise Ratio) gerçek zamanlı hesaplayarak "kategori bias" voltajını otomatik ayarlar.

## 💻 Canlı Etkileşim (Demo)

Ribozom statik bir daktilo değil, tepki veren bir hücredir. Yanlış bildiğini anında düzeltir ve unutmaz:

```text
> kedi ne yer
[HUCRE] once kedi yer .

> hayir, kedi balik yer
[OGRENIYOR] Dogru cevap: 'kedi balik yer'
[LIZOZOM] ' kedi ne yer ?  ' -> 'kedi balik yer .' hafizada.
-> ribo31.bin yazildi (4207 KB)
[OGRENDI] 'kedi balik yer' kaydedildi.

> kedi ne yer
[HUCRE] kedi balik yer .

> köpek nerede yaşar
[HUCRE] ne zaman sever .

> hayir, köpek evde yaşar
[OGRENDI] 'köpek evde yaşar' kaydedildi.

> kedi ne yer
[HUCRE] kedi balik yer .  <-- (Yıkıcı unutma/Catastrophic Forgetting yok!)
```

## 🧠 Mimari Altında Neler Oluyor? (Under the Hood)

Sistem **3700+ satırlık saf C kodunun** modüler bir Unity Build mimarisiyle birleştirilmesinden oluşur:
1. **LCRS Trie (`ribozom_trie.c`):** 10K+ cümlenin $O(N)$ brute-force darboğazını $O(L)$ seviyesine indiren, RAM dostu şablon hafızası.
2. **S-O-V İskeleti (`ribozom_qa.c`):** Özne-Nesne-Yüklem geçiş matrisleri ile dilin mantıksal iskeletini oluşturur.
3. **Golgi Aygıtı (`ribozom_golgi.c`):** Tahmin öncesi kategori bazlı şablon hizalaması.
4. **UTF-8 Byte-Level İşleme:** Karakter kodlamalarını kaba kuvvetle değil, byte dizilimleri üzerinden deterministik olarak işler.

## 🚀 Başlangıç (Getting Started)

Projeyi derlemek için standart bir GCC derleyicisi yeterlidir:

```bash
# Projeyi derle
gcc -O2 -o ribozom ribozom_main.c -lm

# 10K eğitim verisiyle başlat ve interaktif moda geç
./ribozom --interactive
```

## 📜 Felsefe (Neden Ribozom?)
Silikon Vadisi, zekanın ancak milyarlarca parametre ve devasa veri merkezleriyle simüle edilebileceğini savunuyor. Ribozom ise "kütüphanenin" kendisi olmayı değil, o kütüphaneyi anlayan, okuyan ve öğrenen "kütüphaneci" olmayı hedefleyen, bilgisayar biliminin köklerine bir dönüş denemesidir.

***
