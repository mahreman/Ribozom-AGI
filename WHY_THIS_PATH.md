# RIBOZOM — Neden Bu Yol?

## Sektör Aynı Yolu Denedi. Ve Bıraktı.

Sembolik AI (1960–1990) tam bu projenin büyük ölçeğiydi: kural tabanlı sistemler, bilgi grafları, mantık çıkarımı. CYC projesi 40 yıl sürdü, milyonlarca kural elle yazıldı — ölçeklenemedi. OpenCyc, SOAR, ACT-R, NARS — hepsi aynı duvara çarptı. Elle yazılan kurallar gerçek dünyanın karmaşıklığı karşısında yetersiz kaldı.

2012'de deep learning patlama yaptı. 2017'de Transformer mimarisi çıktı, GPT serisi geldi. Endüstri şunu keşfetti: kuralları elle yazmaya gerek yok; yeterli veri ve hesaplama gücüyle sistem kendisi öğreniyor. Ve kısa vadede haklıydılar — GPT-4 insanla konuşabiliyor, kod yazabiliyor, makale özetleyebiliyor.

Peki neden herkes o yolda kaldı?

**Ekonomik teşvik yanlış yönde.** Transformer + GPU = ölçeklenebilir ürün = gelir. Tüm ekosistem bu yöne para akıtıyor. "Küçük başla, mekanizmayı doğru kur, yavaş büyüt" yaklaşımı Wall Street'e satılmaz.

**Akademik teşvik yanlış yönde.** Makale yayınlamak için benchmark'ta SOTA lazım. Transformer'a bir katman ekle, 0.3 puan artır, makale kabul. Sıfırdan yeni mimari kur, küçük ölçekte test et — reviewer "ölçek yetersiz" der.

**Cesaret eksikliği.** PyTorch'u aç, pretrained model yükle, fine-tune et — 2 saatte sonuç var. Saf C'de karakter seviyesinde trie yaz, cluster algoritması tasarla, kausal graf kur — haftalarca sürer, çalışacağı garanti değil. Kurumsal mühendis bu riski almaz.

**Kara kutu "yeterince iyi" görünüyor.** GPT-4 "neden yağmur yağar" sorusuna güzel bir paragraf yazar. Cevap doğru görünür. Ama neden o cevabı verdiğini kimse bilmez. Hallucinate ederse kimse anlamaz. Endüstri bunu kabul etti.

---

## Ribozom'un Farkı

Ribozom farklı bir değer önerisi sunuyor:

> **Transformer "her şeyi biraz bilir." Ribozom "az şey bilir ama bildiklerini kesin bilir ve neden bildiğini açıklayabilir."**

Her kausal edge izlenebilir. Her confidence açıklanabilir. Her çıkarım path'i takip edilebilir. Hallucination kausal katmanda yapısal olarak imkansız — edge yoksa cevap yok, kanıt yoksa çıkarım yok.

**Ne yapıyor:**
- Saf C, sıfır kütüphane, GPU yok, tek süreç
- Karakter seviyesinde öğrenme (Ribozom motoru)
- Distributional clustering ile otomatik kategori keşfi (ER organeli)
- Çift yönlü kausal graf ile nedensellik çıkarımı (Sitoiskelet)
- Polarite aritmetiği ile contrapositive mantık (Hücre Zarı)
- Kısa süreli bellek ile zamir çözümleme (Vesicle)
- Zaman eki tanıma ile temporal tutarlılık (Kronofor)
- Meta-biliş ile "bilmiyorum" yeteneği (Nukleolus)
- Canlı öğrenme ile anlık düzeltme ve kalıcı hafıza (Lizozom)

**Ne yapmıyor:**
- Milyarlarca parametre gerektirmiyor
- GPU gerektirmiyor
- Pretrained model gerektirmiyor
- Framework gerektirmiyor
- Hallucinate etmiyor (kausal katmanda)

---

## Zamanlaması Neden Doğru

Sektör şu anda Transformer'ın sınırlarına çarpıyor. Hallucination çözülemiyor, reasoning zayıf, güvenilirlik düşük, enerji tüketimi devasa. "Daha büyük model = daha iyi" formülü tıkandı. Neuro-symbolic ve hybrid yaklaşımlar yeniden gündemde.

Ribozom bu ortamda küçük ama izlenebilir, güvenilir, hallucination-free bir alternatif inşa ediyor. Ölçek küçük — ama ölçek mühendislik sorunu. Doğru mimariyi bulmak keşif sorunu. Ve keşif yapıldı.

---

## Biyolojik Metafor

Her modül bir hücresel organelin işlevini taklit eder — bu sadece isimlendirme değil, mimari sınır koyar:

| Organel | Biyolojik Karşılık | İşlev |
|---------|-------------------|-------|
| Ribozom | Ribosome | Karakter seviyesi tahmin motoru |
| ER | Endoplasmic Reticulum | Distributional kategori keşfi |
| Mikrotübül | Microtubule | Kategori geçiş matrisi (SOV iskeleti) |
| Golgi | Golgi Apparatus | Cümle bankası ve OOV substitution |
| Lizozom | Lysosome | QA şablon hafızası, canlı öğrenme |
| Sitoiskelet | Cytoskeleton | Çift yönlü kausal graf (dual CSR) |
| Vesicle | Vesicle | Kısa süreli bellek, zamir çözümleme |
| Kronofor | Chronophore | Zaman eki tanıma ve temporal tutarlılık |
| Nukleolus | Nucleolus | Meta-biliş, "bilmiyorum" yeteneği |
| Protein Folding | Protein Folding | Pozisyonel kelime imzası |

---

## Kanıtlar

Dört günde, sıfırdan:

```
"kartal eti yer"                                    — doğru cümle üretimi
"çünkü soğuk oldu . çünkü fırtına oldu ."          — 2-hop kausal zincir + tense
"erken kalkmazsan → geç olur"                       — contrapositive çıkarım
"soğuk olur . sonra kar olur . sonra evde olur ."   — 3-hop forward domino
"bilmiyorum"                                        — meta-biliş (bilmediğini bilme)
"o neden olur → çünkü fırtına"                      — zamir çözümleme + kausal
```

Her çıktının arkasında izlenebilir path, ölçülebilir confidence, belgelenmiş test var. Kara kutu yok.

---

*"Yürürken uçuş planını çizmek." — Ribozom tasarım ilkesi*

*Nisan 2026, Ankara*
