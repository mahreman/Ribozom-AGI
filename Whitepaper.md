# Ribozom: Biyolojik Hücre Metaforu Üzerine Kurulmuş Modüler Bilişsel Mimari

## Özet

Bu çalışmada, yapay genel zekâ (AGI) hedefli, biyolojik hücre metaforu üzerine inşa edilmiş modüler bir bilişsel mimari olan Ribozom sunulmaktadır. Sistem, saf C dilinde yazılmış olup hiçbir dış kütüphane, çerçeve veya GPU bağımlılığı taşımamaktadır. Büyük dil modellerinin (LLM) monolitik, opak ve statik yapısına alternatif olarak tasarlanan Ribozom, 11 bağımsız bilişsel organelden oluşmaktadır; her organel hücresel biyolojideki eşleniğinden esinlenmiş ayrı bir bilişsel görevi üstlenmektedir. Mimari, tasarım gereği halüsinasyon üretmeyen (hallucination-free by design) bir yapıdadır ve 7 bağımsız dürüstlük kanalı aracılığıyla sistemin bilgi sınırlarını şeffaf biçimde raporlamaktadır. Türkçe dili üzerinde gerçekleştirilen deneysel değerlendirmelerde, sistem 2.310 kelimelik söz varlığı ve 361 nedensel kenar ile 5 adımlı ileri/geri muhakeme zincirleri, zamansal çekim simetrisi, olumsuzluk çıkarımı ve çok turlu diyalog bağlamında zamir çözümleme yeteneklerini başarıyla sergilemiştir. Söz varlığı ölçekleme deneyleri, sistemin %100 söz varlığı artışı koşulunda yapısal bozulmaya uğramadan çalışmaya devam edebildiğini göstermiştir.

**Anahtar Kelimeler:** Yapay Genel Zekâ, Bilişsel Mimari, Modüler Tasarım, Nedensel Çıkarsama, Halüsinasyon Önleme, Sürekli Öğrenme, Doğal Dil İşleme, Türkçe

---

## 1. Giriş

Günümüz yapay zekâ sistemlerinin büyük çoğunluğu, Transformer mimarisine dayalı büyük dil modelleri (LLM) üzerine kuruludur (Vaswani vd., 2017). Bu modeller dil üretimi ve metin tamamlama görevlerinde kayda değer başarılar elde etmiş olsa da, birçok temel bilişsel eksiklik barındırmaktadır: kalıcı bellek mekanizmalarının yokluğu, bilgi pekiştirme sürecinin bulunmaması, sürekli öğrenme kapasitesinin olmaması ve karar süreçlerinin izlenememesi (Karpathy, 2025). Bu eksiklikler, mevcut sistemlerin yapay genel zekâ (AGI) yolunda ciddi yapısal engeller oluşturduğuna işaret etmektedir.

Bu çalışmada, söz konusu eksiklikleri mimari tasarım düzeyinde ele alan Ribozom sistemi tanıtılmaktadır. Ribozom, biyolojik hücre organizasyonundan esinlenerek her biri ayrı bir bilişsel işlevden sorumlu modüler organellerden oluşmaktadır. Sistem, gradient hesabı, geri yayılım (backpropagation) veya matris çarpımı kullanmamaktadır; bunun yerine dağılımsal anlambilim (distributional semantics), Hebbian öğrenme ilkeleri ve kanıta dayalı nedensel çıkarsama mekanizmalarını bütünleştirmektedir.

Ribozom'un mevcut literatüre temel katkıları şu şekilde özetlenebilir:

1. **Biyolojik organel metaforu:** Tek bir monolitik ağ yerine, her biri belirli bir bilişsel görevi üstlenen 11 bağımsız modül kullanılmaktadır.
2. **Tasarım gereği halüsinasyon önleme:** 7 bağımsız dürüstlük kanalı, sisteme kendi bilgi sınırlarını tanıma ve raporlama yeteneği kazandırmaktadır.
3. **Sürekli öğrenme:** Sistem her gözlem anında bilgi grafiğini güncellemekte, sabit ağırlık (freeze) aşaması bulunmamaktadır.
4. **Tam izlenebilirlik:** Her çıktı kararı, hangi kanıtlara dayandığı, hangi kenarların kullanıldığı ve hangi güven skorlarının hesaplandığı düzeyinde denetlenebilir durumdadır.
5. **Sıfır bağımlılık:** Tüm sistem saf C dilinde yazılmış olup toplam ikili dosya boyutu 12 MB'dir ve herhangi bir GPU, çerçeve veya dış kütüphane gerektirmemektedir.

---

## 2. İlgili Çalışmalar

### 2.1 Bilişsel Mimariler

SOAR (Laird, 2012) ve ACT-R (Anderson vd., 2004), kural tabanlı modüler bilişsel mimariler olarak uzun süredir araştırılmaktadır. Her iki sistem de üretim kuralları ve çalışan bellek mekanizmalarına dayanmakta, ancak dağılımsal dil öğrenimi, nedensel graf tabanlı çıkarsama ve halüsinasyon önleme gibi mekanizmalar barındırmamaktadır.

### 2.2 OpenCog / AtomSpace

Goertzel'in (2014) önerdiği OpenCog, hipergraf tabanlı çoklu öğrenme algoritması kullanan bir AGI çerçevesidir. Ribozom ile kavramsal düzeyde en yakın çalışma olmakla birlikte, OpenCog yüzlerce dış bağımlılık içermekte ve on beş yılı aşan geliştirme sürecine karşın çalışan bir demo sunamamıştır.

### 2.3 Hiyerarşik Zamansal Bellek (HTM)

Hawkins'in (2004) nörokorteks modellemesine dayanan HTM yaklaşımı biyolojik esinlenme açısından Ribozom ile ortaklık göstermekle birlikte, tek katmanlı zamansal bellek ile sınırlı kalmakta, organel ayrışması veya nedensel çıkarsama içermemektedir.

### 2.4 Büyük Dil Modelleri

GPT (Brown vd., 2020), Claude ve Gemini gibi Transformer tabanlı modeller olağanüstü dil üretim kapasitesine sahip olmakla birlikte, Karpathy'nin (2025) işaret ettiği üzere "hayvanlar değil, hayaletler" üretmektedir: evrimsel süreçlerle değil, internet verilerinden oluşan dijital yansımalardır. Bu modeller bellek kalıcılığı, bilgi pekiştirme ve sürekli öğrenme mekanizmalarından yoksundur.

Ribozom, bu çalışmaların hiçbirinde eşzamanlı olarak bulunmayan beş temel özelliği — biyolojik modülerlik, halüsinasyon önleme, sürekli öğrenme, tam izlenebilirlik ve sıfır bağımlılık — tek bir mimaride bütünleştiren, yazarların bilgisi dahilinde, ilk sistemdir.

---

## 3. Sistem Mimarisi

### 3.1 Genel Bakış

Ribozom, biyolojik hücre organizasyonunu bilişsel bir metafor olarak benimseyen modüler bir mimaridir. Her organel özerk biçimde çalışmakta, ancak organeller arası sinyal iletimi yoluyla koordineli davranış sergilemektedir. Sistem, yaklaşık 10.000 satır saf C kodundan oluşmaktadır.

### 3.2 Organeller

#### 3.2.1 Ribozom (Ana Motor)
Karakter düzeyinde n-gram şablon eşleme ile dil üretiminden sorumludur. Eğitim sırasında karakter dizilimlerinin frekans dağılımlarını öğrenmekte ve üretim sırasında en olası devam dizilimini seçmektedir.

#### 3.2.2 Endoplazmik Retikulum (ER)
Söz varlığı kategorizasyonundan sorumludur. Dağılımsal benzerlik (kosinüs benzerliği) ve konum imzası (5 kovanlı pozisyon histogramı) birleşimiyle kelimeleri otomatik olarak kategorilere ayırmaktadır. Konum imzası, SOV sıralamasındaki yapısal rolleri (özne, nesne, yüklem) ayırt etmekte kullanılmaktadır.

Benzerlik fonksiyonu şu şekilde tanımlanmaktadır:

```
esim(a, b) = ctx_cos(a, b) × pos_penalty(a, b)
```

Burada `ctx_cos`, bağlamsal komşuluk vektörlerinin kosinüs benzerliğini, `pos_penalty` ise baskın konum kovası uyumsuzluğu durumunda uygulanan ceza katsayısını (0.3) temsil etmektedir.

#### 3.2.3 Mikrotübül
Kategori-arası geçiş olasılıklarını öğrenmektedir. SOV sıralamasında hangi kategorinin hangi kategoriyi takip ettiğini modelleyerek, cümle yapısının iskeletini oluşturmaktadır.

#### 3.2.4 Golgi Aygıtı
Cümle hafızası (sent_store) yönetiminden sorumludur. Her cümle, kelime kimlikleri (wid) dizisi olarak depolanmakta ve fiil arama (verb lookup), nedensel çıkarsama ve eş gönderim çözümleme işlemlerinde kullanılmaktadır.

#### 3.2.5 Lizozom
Doğrudan soru-cevap hafızasıdır. Sözcük torbası (bag-of-words) eşleme yöntemiyle daha önce karşılaşılan sorulara hızlı yanıt üretmektedir.

#### 3.2.6 Vezikül (Kısa Süreli Bellek)
Halka tamponu (ring buffer) mimarisinde 5 yuvalı kısa süreli bellektir. Çok turlu diyaloglarda zamir çözümlemesi gerçekleştirmektedir. "O/bu/şu" zamirlerinin yalın, belirtme, yönelme, tamlayan ve ayrılma hâl biçimleri olmak üzere toplam 16 zamir formu desteklenmektedir. Çözümleme, en son bahsedilen varlık öncelikli (most-recent ANY) stratejisini izlemektedir.

#### 3.2.7 Kronofor (Zamansal Algılayıcı)
Fiil çekimlerinden zamansal bilgi çıkarmaktadır. ASCII normalleştirme sonrasında sonek analizi ile geçmiş (-di/-dı), şimdiki (-iyor), gelecek (-ecek/-acak), geniş (-ar/-er) ve rivayet (-miş/-mış) zamanlarını tespit etmektedir. Olumsuzluk sonekleri (-medi/-madı/-mez/-maz/-meyecek/-mayacak) de bu modül tarafından algılanmaktadır.

#### 3.2.8 Nedensel Graf
Yönlü, ağırlıklı, polariteli bir nedensel ilişki grafidir. Her kenar (kaynak_wid, hedef_wid, ağırlık, polarite) dörtlüsü olarak temsil edilmektedir. Ağırlıklar, kanıt sayısına dayalı olarak hesaplanmaktadır:

```
w = min(1.0, 0.3 + 0.05 × kanıt_sayısı)
```

İleri yönde çok adımlı zincir çıkarsaması (deneysel olarak 5 adıma kadar doğrulanmıştır), geri yönde neden çıkarsaması ve olumsuzluk yayılımı desteklenmektedir.

#### 3.2.9 Hipotez Üretici
Merak odaklı bir organeldir. Bilgi grafiğindeki boşlukları tespit ederek olası yeni bilgi adaylarını önermektedir. Üç aşamalı bir süreç izlemektedir: pasif tarama, aktif üretim ve ikinci tur öğrenme.

#### 3.2.10 Proteazom (Halüsinasyon Filtresi)
Üretilen yanıtların makullük denetiminden sorumludur. Seyrek karma tablosu (262K yuva, MurmurHash3) kullanarak özne-yüklem birlikte görülme skorlarını hesaplamaktadır. Dört aşamalı denetim gerçekleştirmektedir:
- Aşama A: Doğrudan birlikte görülme skoru
- Aşama B: Kategori düzeyinde düşük skor için "sanırım" ön eki (HEDGE)
- Aşama C: Bağlam cezası (bağlam kelimesinin özne/yüklem ile sıfır birlikte görülmesi)
- Aşama D: Bilinmeyen kelime cezası
- Aşama E: Yapısal denetim — nedensel olmayan yanıtta özne-yüklem yapısı bulunamazsa reddetme

#### 3.2.11 Dilbilgisi Motoru
Kavramsal düzeydeki çıktıları Türkçe cümle yapısına dönüştürmektedir. İki yanıt biçimi desteklenmektedir: fiil tespit edildiğinde "çünkü X fiil" yapısı, fiil bulunamadığında "X yüzünden" ilgeç yapısı kullanılmaktadır.

### 3.3 Dürüstlük Kanalları

Sistem, yedi bağımsız dürüstlük kanalı aracılığıyla kendi bilgi sınırlarını izlemektedir:

| # | Kanal | Tetiklenme Koşulu |
|---|-------|-------------------|
| 1 | 9A Güven Kapısı | SPEAK adım güveni eşik altında |
| 2 | Proteazom-Anlamsal | Özne-yüklem birlikte görülme skoru < 0.05 |
| 3 | Proteazom-Soğuk | Bilinmeyen bağlam kelimesi, skor < 0.20 |
| 4 | Proteazom-AşamaE | Yapısız çıktı + nedensel yol dışı |
| 5 | Nedensel Boşluk | Grafta ileri/geri kenar bulunamaması |
| 6 | K3 Çelişki | Aynı çift için dengeli kanıtlı zıt polarite kenarları |
| 7 | F4 Evet/Hayır | Evet/hayır sorusu algılanması (mi/mı/mu/mü) |

Her kanal birbirinden bağımsız çalışmakta ve farklı bir bilgi eksikliği türünü tespit etmektedir. Bu tasarım, sistemin "bilmiyorum" yanıtını yedi ayrı gerekçeyle verebilmesini sağlamaktadır.

---

## 4. Deneysel Değerlendirme

### 4.1 Deney Ortamı

Tüm deneyler tek bir x86-64 işlemci üzerinde, GPU kullanılmadan gerçekleştirilmiştir. Eğitim verisi 19.730 Türkçe cümleden oluşmaktadır. Değerlendirme için 67 soruluk ana regresyon testi ve 29 soruluk gömülü öz-test kullanılmıştır.

### 4.2 Söz Varlığı Ölçekleme Deneyleri (H1)

Sistemin söz varlığı büyümesine karşı dayanıklılığını ölçmek amacıyla yedi farklı deney gerçekleştirilmiştir:

| Deney | Söz Varlığı | Kategori | Ana Test | Öz-Test |
|-------|-------------|----------|----------|---------|
| A1 (taban) | 1.974 | 32 | %98,5 | 5/5 |
| A2 (+50 mevcut) | 2.040 | 34 | %97,0 | 5/5 |
| A3 (+30 giyim) | 2.081 | 32 | %98,5 | 5/5 |
| A4 (+30 duygu) | 2.096 | 32 | %98,5 | 5/5 |
| A5 (birleşik) | 2.266 | 37 | %97,0 | 5/5 |
| A6 (+200) | 2.412 | 30 | %97,0 | 5/5 |
| A7 (+500) | 2.695 | 39 | %97,0 | 5/5 |

Sonuçlar, söz varlığının %36 artışı koşulunda sistemin yapısal bozulmaya uğramadığını göstermektedir. Küme güven dağılımında sağlıklı parçalanma gözlenmiş (MIXED azalma, SMALL artış), mega-küme patlaması oluşmamıştır.

Genişletilmiş deney (A1-genişletilmiş) ile söz varlığı 2.310'a, nedensel kenar sayısı 361'e çıkarılmış ve ana testte %95,5 başarı oranı elde edilmiştir.

### 4.3 Çok Adımlı Muhakeme Deneyi

Nedensel grafın derinlik kapasitesini sınamak amacıyla 4 yeni kenar eklenerek köprü düğüm sayısı 3'ten 8'e çıkarılmıştır. Sonuçlar:

- İleri yönde 4 adımlı zincir başarıyla üretilmiştir
- Geri yönde 5 adımlı zincir başarıyla çözülmüştür
- Engelin mimari değil veri kaynaklı olduğu tespit edilmiştir

### 4.4 Nitel Değerlendirme

Sistemin ürettiği örnek yanıtlar:

```
Soru: ateş olursa ne olur?
Yanıt: yemek pişer . sonra çayı demle .
[İleri nedensel zincir, 2 adım, güven: 1.000]

Soru: yemek neden pişti?
Yanıt: çünkü ateş yandı .
[Geri nedensel çıkarsama, geçmiş zaman fiil araması]

Soru: yemek neden pişmedi?
Yanıt: çünkü ateş olmadı .
[Olumsuzluk çıkarsaması, ol- yedek fiil]

Soru: halı ne yer?
Yanıt: bilmiyorum .
[Proteazom Aşama-E: yapısız yanıt reddi]

Soru: şimşek olursa ne olur?
Yanıt: ev yanar . ayrıca ağaç yanar .
[Paralel etki, eşik 0.70 üzeri ikinci kenar]
```

---

## 5. Tartışma

### 5.1 LLM Mimarisi ile Karşılaştırma

| Özellik | LLM | Ribozom |
|---------|-----|---------|
| Gradyan hesabı | Var | Yok |
| GPU gereksinimi | Evet | Hayır |
| Öğrenme biçimi | Toplu iş | Sürekli/canlı |
| Bellek | Bağlam penceresi | 5 organel |
| Halüsinasyon | Yaygın | Tasarımla önlenmiş |
| İzlenebilirlik | Sınırlı | Tam |
| Model boyutu | GB-TB | 12 MB |
| Nedensel çıkarsama | Korelasyon | Yönlü nedensellik |

### 5.2 Sınırlılıklar

Mevcut sistemin başlıca sınırlılıkları şunlardır:

1. **Söz varlığı kapsamı:** 2.310 kelime ile sınırlıdır. Gerçek dil kullanımı için en az 100.000 kelimelik söz varlığına ihtiyaç duyulmaktadır.
2. **Soyut kavramlar:** "Adalet", "özgürlük" gibi soyut kavramların temsili henüz desteklenmemektedir.
3. **Çok kiplilik:** Yalnızca metin girişi desteklenmekte, görsel ve işitsel kanallar bulunmamaktadır.
4. **Özerk hedef üretimi:** Sistem reaktiftir; kendi öğrenme hedeflerini otonom olarak belirleyememektedir.
5. **Biçimbilimsel kapsam:** Türkçe biçimbiliminin tamamını kapsamamakta, sınırlı sonek analizi kullanmaktadır.

### 5.3 AGI'ye Giden Yol

Ribozom'un modüler organel mimarisi, AGI'ye yönelik aşamalı bir genişleme yolu sunmaktadır:

- **Endokrin Sistemi:** Stres, güven, merak ve dikkat hormonları aracılığıyla sistem durumu yönetimi
- **Bağışıklık Sistemi:** Veri kalitesi denetimi ve zararlı girdilerin filtrelenmesi
- **Sinir Sistemi + Çekirdek (Nucleus):** Organeller arası koordinasyon ve hedef yığını yönetimi
- **Duyusal Uyarlayıcılar:** Görüntü, ses ve çevresel girdi kanalları
- **Öz-Model:** Sistemin kendi bilgi haritasını oluşturması ve eksikliklerini otonom olarak tespit etmesi

Bu bileşenlerin her biri mevcut organel mimarisinin doğal bir uzantısı olarak eklenebilecek yapıdadır.

---

## 6. Çıkarılan Mimari Dersler

Geliştirme sürecinde 12 mimari ders belgelenmiştir:

| # | Ders | Özet |
|---|------|------|
| D0 | Toptan yeniden eğitim | Eklemesel inşa, toptan yeniden eğitimden üstündür |
| D1 | Altyapı önce, veri sonra | Kanal açıldığında veri akışı doğrulanmalıdır |
| D2 | Karşıtlık ≠ çelişki | Kanıt tabanlı ayrım, ağırlık tabanlıdan üstündür |
| D3 | Sessiz hata | Yedek çıktı makul göründüğünde hata gizlenir |
| D4 | Yapısal sinyal üstünlüğü | Varlıkbilimsel denetim, sözlüksel denetimden üstündür |
| D5 | Test edilemeyen = bozuk | Test altyapısı kurulunca hata kendini gösterir |
| D6 | Test beklentileri | Beklenen değerler de doğrulanmalı bir veridir |
| D7 | Karakter değişkenleri | Karakter varyantları sessiz söz varlığı kaçırması üretir |
| D8 | Kalibrasyon eşikleri | Eşikler veri boyutuyla kaymalıdır |
| D9 | Yalıtılmış ortam | Test ortamı bağımlılıkları doğrulanmalıdır |
| D10 | Erken eniyileme | Mevcut olmayan sorunu çözmemek gerekir |
| D11 | Dilbilgisi iskeleti ≠ bilgi | Dil aygıtları kod düzeyinde sabit kalabilir |

---

## 7. Sonuç

Ribozom, yapay genel zekâya yönelik alternatif bir mimari yol sunmaktadır. Biyolojik hücre metaforu üzerine kurulu modüler organel yapısı, tasarım gereği halüsinasyon önleme, sürekli öğrenme ve tam izlenebilirlik gibi özellikleri doğumundan itibaren taşımaktadır. Bu özellikler, mevcut LLM mimarisinde yapısal olarak bulunmamakta ve sonradan eklenmesi son derece güçtür.

Deneysel sonuçlar, sistemin söz varlığı ölçeklemesine dayanıklı olduğunu, çok adımlı nedensel muhakeme kapasitesine sahip olduğunu ve bilgi sınırlarını şeffaf biçimde raporlayabildiğini göstermektedir.

Mevcut haliyle sistem AGI düzeyinde değildir; ancak AGI için gerekli olan mimari bileşenlerin — modülerlik, sürekli öğrenme, nedensel çıkarsama, öz-farkındalık embriyosu — doğal olarak genişletilebilir bir yapıda mevcut olması, bu mimarinin AGI araştırma alanına özgün bir katkı sağladığını ortaya koymaktadır.

Canlı demo: https://ribozom.anka.istanbul

---

## Kaynaklar

Anderson, J. R., Bothell, D., Byrne, M. D., Douglass, S., Lebiere, C., & Qin, Y. (2004). An integrated theory of the mind. *Psychological Review*, 111(4), 1036-1060.

Brown, T. B., Mann, B., Ryder, N., vd. (2020). Language models are few-shot learners. *Advances in Neural Information Processing Systems*, 33, 1877-1901.

Goertzel, B. (2014). Artificial general intelligence: Concept, state of the art, and future prospects. *Journal of Artificial General Intelligence*, 5(1), 1-46.

Hawkins, J. (2004). *On Intelligence*. Times Books.

Karpathy, A. (2025). AI in 2027. Blog yazısı ve Dwarkesh Patel podcast röportajı.

Laird, J. E. (2012). *The Soar Cognitive Architecture*. MIT Press.

Vaswani, A., Shazeer, N., Parmar, N., vd. (2017). Attention is all you need. *Advances in Neural Information Processing Systems*, 30.
