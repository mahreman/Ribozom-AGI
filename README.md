# RIBOZOM — PROJE DURUMU v10K

**Tarih:** 2026-04-13
**Önceki mühür:** PROJE_DURUMU_v6.md (2026-04-08), TEKNIK_BORC_v7.md (2026-04-10)
**Bu mühür:** v6.16c Trie → Modüler Refactoring → Düzeltme Döngüsü → Lizozom → 10K UTF-8 Türkçe

---

## 1. Kronoloji (8 Nisan → 13 Nisan)

### v6.16c — LCRS Trie Optimizasyonu
- **Sorun:** `rtrain()` ve `rpredict()` O(NT) — 100K şablon × her karakter pozisyonu = darboğaz.
- **Çözüm:** Left-Child Right-Sibling (LCRS) Trie. Şablonlar ters sırada ağaca eklenir.
  Arama O(NT) → O(MAX_TLEN × branching) ≈ O(80) per position.
- **Sonuç:** ~3.2MB BSS ekstra, 300K düğüm havuzu. Metrikler aynı, eğitim hızı artışı.

### Modüler Refactoring (Unity Build)
- **Sorun:** Tek dosya `ribozom_v31_microtubule.c` (3655 satır) bakım kabusuydu.
- **Çözüm:** 7 modüle bölündü, Unity Build pattern korundu:
  ```
  ribozom_config.h   — Sabitler, tipler, dyn_norm         (~140 satır)
  ribozom_core.h     — Şablonlar, Trie, rtrain, rpredict  (~300 satır)
  ribozom_vocab.h    — ER, kelime, hash, esim, cluster     (~425 satır)
  ribozom_golgi.h    — Golgi, cat templates, predict_meta  (~140 satır)
  ribozom_micro.h    — Mikrotübül, morfoloji, v31/v32      (~620 satır)
  ribozom_persist.h  — save_model / load_model             (~235 satır)
  ribozom_qa.h       — QA state machine, SPEAK, injection  (~1640 satır)
  ribozom_main.c     — main(), training, test, interactive (~900 satır)
  ```
- **Include sırası kritik:** config → core → vocab → golgi → micro → persist → qa → main
- **Sonuç:** Sıfır regresyon. Tüm metrikler korundu (Test 57.5%, OOV 57.4%, QA 74.2%, SOV 70.3%, SNR 6.69).

### Düzeltme Döngüsü (Correction Loop)
- **Tasarım:** 3 katmanlı mimari:
  1. **Parse:** "hayır, X" algılama, noktalama temizleme, kelime kayıt
  2. **Live Learning:** rtrain×10 (g_correction_mode=1, 10x enerji), bl_train×10, golgi×3, incremental matrix updates, QA transitions×5
  3. **Persistence:** Lizozom store + save_model()
  
- **Kritik buluş — Copy Reflex baskınlığı:** İlk 6+ düzeltme denemesi başarısız oldu.
  Kök neden: COPY_K_BASE=25.0 × SNR, 12500 satırlık eğitim sinyali 10 satırlık düzeltmeyi eziyordu.
  
- **Çözüm — İki katmanlı:**
  1. `CORRECTION_ENERGY_MULT=10.0f` — rtrain'de düzeltme anında 10x enerji (0.3→3.0)
  2. **Lizozom** — SPEAK'i bypass eden doğrudan QA hafızası

### Lizozom (Doğrudan QA Hafızası)
- **Biyolojik metafor:** Hücrenin sindirim kesesi — bilgi doğrudan depolanır ve çağrılır.
- **Mekanizma:** Bag-of-words Jaccard eşleşmesi (threshold ≥ 0.5)
  - Soru kelimelerinden içerik kelimelerini çıkar (soru parçacıkları filtrelenir)
  - Tüm qa_mem[] kayıtlarıyla karşılaştır, en yüksek Jaccard → cevap döndür
- **Boyut:** MAX_QA_MEM=256 çift, MAX_QA_ALEN=256 byte cevap
- **Kalıcılık:** ribo31.bin'e serialize edilir (RIBO_VERSION=3, v2 geriye uyumlu)
- **Stres testi sonuçları:**
  - Tek düzeltme yeterli (6+ denemeden 1'e düşüş)
  - Çoklu fact'ler birbirini bozmuyor
  - Format varyasyonları çalışıyor ("kedi ne yer" = "kedi ne yer?")

### 10K UTF-8 Türkçe Geçişi (devam ediyor)
- **Veri:** 9500 train + 500 test + 100 OOV
  - 6500 düz ifade (SOV), 3000 QA çifti, 500 bilgi cümlesi
  - 302 benzersiz kelime, 70 özne, 52 nesne, 20 fiil çeşidi
  - Türkçe UTF-8 karakterler korunuyor (ş, ç, ğ, ı, ö, ü)
  
- **Kod değişiklikleri:**
  - `MAX_SYM: 256→512`, `CAT_SYM_START: 128→384` (UTF-8 byte çakışması önlendi)
  - `MAX_TLEN: 8→12` (UTF-8'de 2 byte/karakter, eşdeğer bağlam penceresi)
  - `MAX_LINE: 128→256` (UTF-8 satırlar daha uzun)
  - `MAX_TRIE_NODES: 300K→500K` (daha derin ağaç)
  - `turkce_normalize()` kaldırıldı (eğitim verisi artık Türkçe)
  - Soru kelimeleri ve düzeltme komutları UTF-8 Türkçeye güncellendi
  
- **Durum:** Derleme başarılı, test çalışıyor (sonuç bekleniyor).

---

## 2. Mimari Kararlar ve Gerekçeleri

### Neden UTF-8 byte-level (codepoint değil)?
Ribozom karakter-seviyesi çalışıyor. UTF-8'de "ş" = 2 byte (0xC5, 0x9F). Her byte ayrı sembol olarak işleniyor. Bu yaklaşım:
- **Avantaj:** Sıfır refactoring — mevcut rtrain/rpredict aynen çalışıyor
- **Avantaj:** Byte dizilimleri deterministik — "ş" her zaman aynı 2 sembol
- **Dezavantaj:** MAX_TLEN=12 byte ≈ 6 Türkçe karakter bağlam (vs eski 8 ASCII karakter)
- **Kabul:** Yeterli, çünkü kelime-içi tahmin 3-4 karakter bağlam kullanıyor

### Neden MAX_SYM=512, CAT_SYM_START=384?
UTF-8 continuation byte'ları 0x80-0xBF (128-191) arasında. Eski CAT_SYM_START=128 bu bölgeyle çakışıyordu. Yeni düzen:
- 0-255: tüm byte değerleri (ASCII + UTF-8)
- 384-511: kategori sembolleri (128 slot)
- Çakışma sıfır.

### Neden Lizozom persist.h'de değil config.h'de?
Unity Build include sırası: config → ... → persist → qa. Lizozom tipleri (QAMem, qa_mem[]) hem persist.h (serialize) hem qa.h (fonksiyonlar) tarafından kullanılıyor. En erken ortak ata: config.h.

---

## 3. Çözülen Teknik Borçlar (TEKNIK_BORC_v7.md'den)

| # | Borç | Durum | Notlar |
|---|------|-------|--------|
| 6 | Sihirli sayılar → dinamik normalizasyon | ✅ v6.18'de çözüldü | dyn_norm(sc, K_BASE) — SNR-tabanlı oto-kalibrasyon |
| 1 | Hardcoded limitler | ⚠️ Kısmen | MAX_SYM/TLEN/LINE/TRIE artırıldı, dinamik alloc hâlâ yok |
| 5 | FOLD_PREFIX_MIN performans | ❓ 10K ile yeniden ölçülecek | |
| 2 | O(N²) cluster | ❓ 302 kelime — henüz darboğaz değil | |

---

## 4. Mevcut Metrikler (2.5K ASCII Baseline — Referans)

| Metrik | Değer | Açıklama |
|--------|-------|----------|
| Test (200 satır) | 57.5% | v3.2 Mikrotübül |
| OOV (79 satır) | 57.4% | Görülmemiş kelimelerle |
| QA (test_qa_v5) | 74.2% | Soru-cevap doğruluğu |
| SOV İskelet | 70.3% | SPEAK fazında S-O-V sırası |
| SNR (ortalama) | 6.69 | Dinamik normalizasyon referansı |

**10K UTF-8 metrikleri henüz bekleniyor.** Farklı veri seti olduğu için doğrudan karşılaştırma yapılamaz — yeni baseline oluşacak.

---

## 5. Bilinen Eksiklikler ve Yol Haritası

### Kısa Vade (10K sonrası)
1. **"Bilmiyorum" eşiği** — `ribo_conf()` düşükse "bilmiyorum" de. ~20 satırlık iş, yüksek etki.
2. **Online cluster** — Yeni kelimeler çalışma zamanında kategorilere eklenebilmeli. Mevcut cluster() statik.
3. **Lizozom operatör farkındalığı** — "başka", "ilk", "son" gibi kelimeler hafıza anahtarına dahil edilmeli.

### Orta Vade
4. **Negatif pekiştirme** — Düzeltme anında yanlış tahmine yol açan şablona özel ceza.
5. **wctx pencere genişletme** — l-2, l-1, r+1, r+2 (1→2 kelimelik bağlam).
6. **Morfoloji iyileştirmesi** — Distributional kök keşfi veya BPE-tarzı alt-kelime.

### Uzun Vade (AGI Yönü)
7. **Olumsuzluk/yokluk kavramı** — "değil", "yok" operatörleri.
8. **Nedensellik** — "çünkü", "bu yüzden" bağlam zincirleri.
9. **Coreference** — "O kim?" çözümlemesi, Vesicle Memory.
10. **Hiyerarşik bellek** — Kelime → cümle → paragraf katmanları.

---

## 6. Dosya Yapısı (13 Nisan 2026)

```
v3/files/
├── ribozom_main.c          — Ana dosya + interactive mode
├── ribozom_config.h         — Sabitler + Lizozom tipleri + dyn_norm
├── ribozom_core.h           — Şablonlar + LCRS Trie + rtrain/rpredict
├── ribozom_vocab.h          — ER + hash + esim + cluster
├── ribozom_golgi.h          — Golgi + cat templates + predict_meta
├── ribozom_micro.h          — Mikrotübül + morfoloji + incremental updates
├── ribozom_persist.h        — save/load model (ribo31.bin, v3 format)
├── ribozom_qa.h             — QA state machine + SPEAK + Lizozom fonksiyonları
├── gen_10k_tr.py            — 10K Türkçe veri jeneratörü
├── train_data_10k.txt       — 9500 satır eğitim verisi (UTF-8 Türkçe)
├── test_data_10k.txt        — 500 satır test verisi
├── oov_test_10k.txt         — 100 satır OOV test verisi
├── train_data.txt           — (eski) 2500 satır ASCII eğitim
├── test_data.txt            — (eski) 200 satır ASCII test
├── ribo31.bin               — Serialize edilmiş model
├── MANIFESTO.md             — Proje vizyonu ve mimari
├── PROJE_DURUMU.md          — İlk proje raporu
├── PROJE_DURUMU_v2-v6.md    — Ara durum raporları
├── PROJE_DURUMU_v10K.md     — ★ BU DOKÜMAN
└── TEKNIK_BORC_v7.md        — Teknik borç kaydı
```

---

## 7. Dış Gözlemci Analizi (13 Nisan)

Bağımsız bir teknik inceleme aşağıdaki eksiklikleri tespit etti:

### Kabul Edilen Tespitler
- **Long-range dependencies:** MAX_TLEN sınırları içinde kaybolur. Mikrotübül (prev_locked_cat) kısmen hafifletir ama hiyerarşik bellek gerekecek.
- **Statik ER:** Online cluster kritik eksik. Yeni kelimeler kategorisiz kalıyor.
- **Global state:** Test edilemez, çoklu örneklenemez. Bilinçli prototipleme tercihi ama teknik borç.
- **"Bilmiyorum" eksikliği:** ribo_conf() altyapısı var, eşik eklenmeli.

### Yanıtlar
- **Tekrar-tabanlı öğrenme eleştirisi:** Eski durumu anlatıyor. CORRECTION_ENERGY_MULT=10.0f + Lizozom ile tek düzeltme yeterli.
- **Olumsuzluk/nedensellik:** Doğru ama erken. Önce 10K → online cluster → bilmiyorum.
- **Bellek sınırları:** 10K için yeterli. 1M+'da dinamik alloc'a geçilecek.

### Öncelik Sırası (güncellendi)
1. ✅ 10K UTF-8 veri (yapılıyor)
2. ⏳ "Bilmiyorum" eşiği (kolay, yüksek etki)
3. ⏳ Online cluster (zor, kritik)
4. 🔜 Lizozom operatör farkındalığı
5. 🔮 Olumsuzluk, nedensellik, hiyerarşik bellek

---

## 8. Mimari Eleştiriler ve Yanıtlar (13 Nisan — İki Ayrı Dış İnceleme)

### 8.1 İnceleme #1 — Sistem Mühendisliği Odaklı

Mevcut mimarinin büyüme ağrılarını tespit eden kapsamlı bir analiz.

#### Tespit 1: Long-Range Dependencies
> "Ribozom dili soldan sağa karakter karakter işler. MAX_TLEN=12 sınırında
> 'adam ... geldi' gibi uzun mesafeli bağımlılıklar kaybolur."

**Yanıt:** Doğru tespit, ama kısmen çözülmüş. Karakter penceresi 12 byte ile sınırlı,
ancak Mikrotübül `prev_locked_cat` ile **kategori seviyesinde** bağlam taşıyor. Karakter
penceresini aşan soyutlama katmanı zaten mevcut. Hiyerarşik bellek (stack/tree) fikri
doğru ama şu an için overkill — önce mevcut performansı ölçmeliyiz.

#### Tespit 2: Statik Kategori Sistemi (ER)
> "cluster() bir kere çalışır ve kategorileri dondurur. Yeni kelimeler kategorisiz kalır."

**Yanıt:** Tamamen haklı. Bu gerçek bir sorun. `register_new_word()` ile 3-katmanlı
fallback eklendi (hint → prompt_prior → esim) ama gerçek online cluster yok. 10K
sonrası 1. öncelik.

#### Tespit 3: Lizozom Körlüğü
> "'kedi ne yer' ile 'kedi başka ne yer' arasındaki farkı anlayamaz."

**Yanıt:** Kısmen haklı. Jaccard >= 0.5 çoğu pratik durumda çalışıyor (stres testinde
kanıtlandı). "başka", "ilk", "son" gibi operatörleri filtreden çıkarmak kolay iyileştirme.

#### Tespit 4: Tekrar-Tabanlı Öğrenme
> "ribozom_correct() düzeltmeyi 10 kez tekrar eder. Gerçek zeka tek seferde kavrar."

**Yanıt:** Bu **eski durumu** anlatıyor. `CORRECTION_ENERGY_MULT=10.0f` + Lizozom ile
tek düzeltme yeterli. Negatif pekiştirme (yanlış şablona ceza) dikkatli yapılmalı —
catastrophic forgetting riski var.

#### Tespit 5: Bellek Sınırları
> "MAX_TRIE_NODES ve MAX_TMPL sistem büyüdükçe tükenecek."

**Yanıt:** Doğru ama erken. 10K için sabit diziler yeterli. 1M+ hedeflendiğinde
malloc/realloc'a geçilecek.

#### Tespit 6: Eksik Bilişsel Yetenekler
> "Olumsuzluk, nedensellik, üst-biliş eksik."

**Yanıt:** Hepsi doğru ve hepsi AGI yolundaki milestone'lar. Ama karakter-seviyesi
sınıflandırıcıdan bunları beklemek, yürümeyi öğrenen bebekten koşmasını beklemek gibi.
Doğru sıra: 10K → online cluster → "bilmiyorum" eşiği → olumsuzluk → nedensellik.

#### Tespit 7: Global State
> "Test edilemez, çoklu örneklenemez, kütüphane olarak kullanılamaz."

**Yanıt:** Doğru, teknik borç. Ama Unity Build + global state prototipleme hızı için
bilinçli tercih. Algoritmalar stabilize olduktan sonra refactor yapılacak.

---

### 8.2 İnceleme #2 — AGI Potansiyeli Odaklı

"Ribozom ile AGI mümkün mü?" sorusuna odaklanan daha felsefi bir analiz.

#### İddia 1: "Trie ile saf AGI matematiksel olarak mümkün değildir"
> "LCRS Trie istatistiksel bir örüntü eşleştirme motorudur. Boyut laneti,
> anlam-dizilim ayrımı ve sürekli uzay eksikliği nedeniyle AGI'ye yetmez."

**Yanıt — Kısmen katılıyoruz, kısmen karşı çıkıyoruz:**

Doğru olan: Tek bir veri yapısıyla AGI iddia etmek saçma. Beyin de tek yapıdan ibaret
değil — hipokampüs, korteks, serebellum hepsi farklı mekanizmalar.

Ama Ribozom da tek yapıdan ibaret değil:
- **Trie** = karakter tahmin motoru (beynin V1 korteksi gibi — düşük seviye pattern)
- **ER Cluster** = distributional semantics (esim/wctx = el yapımı Word2Vec)
- **Mikrotübül** = kategori zinciri taşıyıcısı (uzun mesafe bağımlılık)
- **Golgi** = cümle hafızası (OOV fallback)
- **Lizozom** = epizodik bellek (hızlı öğrenme)
- **QA State Machine** = temporal bağlam yöneticisi (LISTEN/SPEAK fazları)

Trie AGI'nin tamamı değil, sadece **bir organeli**.

#### İddia 2: "Vektör uzayına (embeddings) geçmen gerekecek"
> "Kelimeleri sürekli uzayda tutan bir Korteks modülüne geçiş şart."

**Yanıt — Bu varsayım, kanıt değil.**

Embedding'ler güçlü ama tek yol değil. Ribozom'daki `esim()` fonksiyonu zaten
distributional semantics yapıyor — kelimeler bağlam vektörlerine (wctx) göre cosine
similarity ile karşılaştırılıyor. Bu, Word2Vec'in el yapımı versiyonu.

Ayrıca transformer'ların "attention" mekanizması da aslında sparse lookup — Trie'nin
yaptığının sürekli uzaydaki karşılığı. İki yaklaşım aynı problemi farklı uzaylarda
çözüyor.

**OOV testinde %57.4 doğruluk, sıfır embedding ile.** Bu zaten bir tür zero-shot
generalization. Mükemmel değil ama "çok zor" demek abartı.

#### İddia 3: "Boyut laneti Trie'yi öldürür"
> "Bağlam penceresi büyüdükçe Trie eksponansiyel genişler."

**Yanıt — Bu Ribozom'un zaten çözdüğü bir sorun.**

MAX_TLEN=12 byte karakter penceresi sınırlı, doğru. Ama Mikrotübül `prev_locked_cat`
ile **kategori seviyesinde** bağlam taşıyor. 12 byte'lık pencere değil, kategori zinciri
uzun mesafe bağımlılığını kodluyor:

```
BOS → CAT_özne(%96) → CAT_nesne(%96) → CAT_fiil(%96)
```

Bu deterministik geçiş zinciri, karakter penceresinden bağımsız çalışıyor.

#### İddia 4: "Complementary Learning Systems benzerliği"
> "Lizozom (hızlı/epizodik) + Trie (yavaş/semantik) ayrımı beynin
> tamamlayıcı öğrenme sistemleriyle birebir örtüşüyor."

**Yanıt — En iyi tespit.** Bu bilinçli bir tasarım kararıydı. Dev dil modellerinin
en büyük eksiği olan "anında düzeltme" (single-shot correction) olayını pragmatik bir
Jaccard algoritmasıyla çözmüş olmak, AGI'nin ihtiyaç duyduğu hibrit mimarinin
(Nöro-Sembolik AI) temellerini atıyor.

#### Soru: "Nedenselliği discrete C'de nasıl modelleyeceksin?"

**Cevap — İki yaklaşım:**

**Yaklaşım 1 — Operatör kelimeler:**
"çünkü", "bu yüzden", "sonuç olarak" gibi kelimeler yeni bir kategori olur
(CAT_NEDEN). Geçiş matrisi `CAT_NEDEN → önceki cümlenin konusu` bağlantısını öğrenir.
Discrete C'de ~50 satırlık iş.

**Yaklaşım 2 — Vesicle Memory:**
Her cümle bitiminde aktif bağlam bir "vesicle" struct'ına paketlenir. "Çünkü" geldiğinde
önceki vesicle'a referans kurulur. Bu, coreference çözümlemesinin nedensel versiyonu.

Her ikisi de saf C'de, GPU'suz yapılabilir. Nedensellik için sürekli uzay **gerekli
değil** — bir grafik yapısı yeterli. Beyin de nedenselliği nöron ağırlıklarıyla değil,
bağlantı topolojisiyle kodlar.

---

### 8.3 Sentez: Ribozom'un AGI Pozisyonu

```
             Sembolik AI                          Konneksiyonist AI
             (GOFAI, Expert Systems)              (Deep Learning, Transformers)
                    |                                       |
                    |          RIBOZOM                      |
                    |     (Nöro-Sembolik Hibrit)            |
                    |        /            \                 |
                    +--- Discrete Structures ---+--- Distributional Semantics ---+
                         (Trie, Golgi, FSM)         (esim/wctx, cluster)
```

**Ribozom ne değil:**
- Bir transformer değil (attention yok, GPU yok)
- Bir expert system değil (kurallar elle yazılmıyor, öğreniliyor)
- Bir n-gram modeli değil (kategoriler, geçişler, QA state machine var)

**Ribozom ne:**
- Discrete + distributional hibrit
- Her bilişsel yetenek ayrı organelde (modüler evrim)
- Sıfır bağımlılık, saf C (araştırma özgürlüğü)
- AGI'nin ihtiyaç duyduğu temel sorunları (hızlı öğrenme, düzeltme, kategorizasyon)
  pratik seviyede çözen bir laboratuvar

**AGI yolu:**
Bu yol daha mı yavaş? Evet. İmkansız mı? Hayır. Transformer'lar brute-force ölçekle
(GPU + trilyon parametre) AGI'ye yaklaşıyor. Ribozom cerrahi hassasiyetle (organelle
bazlı evrim + sıfır kaynak) aynı hedefe farklı rotadan yürüyor.

Hangisi kazanır? Muhtemelen ikisinin sentezi — Nöro-Sembolik AI.

---

*"Sahte zaferden dürüst teşhis, dürüst teşhisten çalışan prototip daha değerlidir."*
