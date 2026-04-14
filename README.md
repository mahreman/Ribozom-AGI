# RIBOZOM — PROJE DURUMU v15K (7 Domain)

**Tarih:** 2026-04-13
**Önceki mühür:** PROJE_DURUMU_v10K.md (2026-04-13), TEKNIK_BORC_v7.md (2026-04-10)
**Bu mühür:** 10K Stabilizasyon → Kritik Bug Fix'ler → Stress Test → 15K 7-Domain Genişleme

---

## 1. Kronoloji (13 Nisan — Tek Gün İçi)

### 1.1 prompt_prior_qa Serialization Gap — Kritik Fix
- **Sorun:** `--train` olmadan load-only modda QA SPEAK çöp üretiyordu ("ke¿aek¿").
  Kök neden: `prompt_prior_qa[][]` matrisi ribo31.bin'e serialize edilmiyordu.
  Load sonrası QA state machine tamamen boş — q_dom hesaplanamıyor, inject_qa_prior tetiklenmiyor.
- **Denenen çözüm:** prompt_prior_qa'yı persist.h'e eklemek → Unity Build sırası hatası
  (prompt_prior_qa, qa.h'de tanımlı, persist.h qa.h'den önce include ediliyor). Revert edildi.
- **Uygulanan çözüm:** Load-only modda "Hızlı Faz 6 Rebuild" — train verisini yükleyip
  SADECE `build_soft_cats()` + `v5_init_token_ids()` + `build_qa_transitions()` çalıştır.
  - Süre: **0.01 saniye** (65 dakikalık tam eğitim yerine)
  - QA Metrik 1: 49% → **69.8%**
  - SPEAK: çöp → gerçek cümleler
  - q_dom hesaplanan: 0 → 350/350

### 1.2 Tekrarlı Öğrenme İdempotency Fix
- **Sorun:** Aynı düzeltme tekrar verildiğinde `ribozom_correct()` her seferinde
  rtrain×10 + golgi×3 + bl_train×10 çalıştırıyordu → şablon şişmesi.
  5 tekrarda model 10.6MB → 11.9MB, şablon sayısı sürekli artıyordu.
- **Çözüm:** `ribozom_correct()` başında Lizozom idempotency kontrolü eklendi:
  ```c
  int existing = qa_mem_lookup(question);
  if (existing >= 0 && strcmp(qa_mem[existing].answer, test_ans) == 0) {
      qa_mem[existing].hit_count++;  // Sadece sayaç artır
      save_model("ribo31.bin");
      return;  // Ağır eğitim ATLA
  }
  ```
- **Sonuç:** 5 tekrarda şablon sayısı ve model boyutu **tamamen sabit**. Sıfır şişme.
  Farklı cevap geldiğinde tam eğitim yolu korunuyor.

### 1.3 Lizozom Stress Test — 10 Test, 9 PASS, 0 FAIL
Otomatik stress test scripti (stress_test.sh) 10 senaryo:

| # | Test | Sonuç | Detay |
|---|------|-------|-------|
| 1 | Temel Öğrenme (20 bilgi) | ✅ PASS | 20/20 öğrenildi |
| 2 | Geri Çağırma | ✅ PASS | 20/20 doğru hatırlandı |
| 3 | Persistence (kapat-aç) | ✅ PASS | 6/6 doğru |
| 4 | Çakışma (güncelleme) | ✅ PASS | balık→tavuk güncellendi |
| 5 | Operatör Farklılığı | ✅ PASS | "ne yer"→tavuk, "ne içer"→süt AYRI |
| 6 | Tekrarlı Öğrenme | ✅ PASS | Sıfır şişme (fix sonrası) |
| 7 | Toplu Yükleme (30 bilgi) | ✅ PASS | 30/30, toplam 47 lizozom |
| 8 | Karışık Geri Çağırma | ✅ PASS | 5/6 (1 miss: bag-of-words çakışma) |
| 9 | Bilinmeyen Soru (SPEAK fallback) | ✅ PASS | SPEAK devreye giriyor |
| 10 | Kapasite Analizi | ✅ PASS | 47/256 slot (%82 boş) |

**Kritik bulgu:** Operatör körlüğü YOK! "kedi ne yer" ≠ "kedi ne içer" çünkü "yer"/"içer"
soru filtre listesinde değil → Jaccard farklı kelime setleri olarak algılıyor.

**T8 miss analizi:** "köpek nasıl ses çıkarır" → "havhav" yerine "öööö" döndü.
"ses çıkarır" ortak kelimeler tüm hayvan sesi sorularında aynı → son kaydedilen kazanıyor.
Çözüm: TF-IDF veya pozisyon ağırlığı (ileride).

### 1.4 Hayır Fix Doğrulaması
İnteraktif mod tam çalışma döngüsü test edildi:
- ✅ UTF-8 "hayır" (ı harfi) doğru algılanıyor
- ✅ Virgül yapışması yok ("hayır, balık" → cevap temiz "balık")
- ✅ İlk seferde algılama (önceki bug: ilk hayır atlanıyordu)
- ✅ "hhayir" karakter ikileşmesi giderildi
- ✅ Düzeltme sonrası Lizozom'dan doğru geri çağırma

### 1.5 15K 7-Domain Veri Jeneratörü
Mevcut 10K hayvan-yemek corpus'unun ötesine geçmek için 7 domain'li jeneratör yazıldı:

| Domain | Özne Örnekleri | Nesne Örnekleri | Fiil Örnekleri | Cümle Sayısı |
|--------|---------------|-----------------|----------------|-------------|
| D0: Hayvan-Yemek | kedi, aslan, çiftçi | balığı, ekmeği | yer, içer, sever | 5000 |
| D1: Meslek-İş | doktor, avukat, mimar | hastayı, projeyi | tedavi eder, tasarlar | 1600 |
| D2: Spor | futbolcu, yüzücü, atlet | topu, kupayı | atar, koşar, kazanır | 1300 |
| D3: Aile-Günlük | anne, baba, bebek | yemeği, hikâyeyi | pişirir, anlatır | 1300 |
| D4: Doğa-Hava | yağmur, güneş, nehir | toprağı, ağaçları | ıslatır, yağar* | 1300 |
| D5: Bilim-Eğitim | fizikçi, biyolog | deneyi, atomu | inceler, keşfeder | 1300 |
| D6: Ulaşım | şoför, otobüs, tren | yolcuyu, yükü | taşır, sürer, uçar | 1300 |
| Cross-domain köprüler | — | — | — | 500 |

*D4 özel: SV (nesnesiz) cümleler dahil — "yağmur yağar", "rüzgâr eser"

**İstatistikler:**
- Toplam train: **13600** satır (8657 ifade + 4943 QA)
- Test: **500** satır (7 domain'den temsil)
- OOV: **100** satır (3 domain'den bilinmeyen kelimeler)
- Benzersiz kelime: **782** (10K'daki 306'dan 2.5x artış)
- SV cümleler: **~1064** (SOV dışı yapısal çeşitlilik)
- Cross-domain köprüler: **~500** ("anne kediyi sever" gibi domain-arası cümleler)

**Eğitim durumu:** ✅ Tamamlandı (68 dk). v7.1 stabilize edildi.

---

## 2. Güncel Metrikler

### 2.1 10K UTF-8 Baseline (Referans)

| Metrik | 2.5K ASCII | 10K UTF-8 | Fark | Yorum |
|--------|-----------|-----------|------|-------|
| Test | 57.5% | 63.1% | +5.6 | Gerçek ilerleme |
| OOV | 57.4% | 46.4% | -11.0 | Daha zor OOV seti |
| QA Metrik 1 | 74.2% | 69.8% | -4.4 | Daha geniş soru çeşitliliği |
| Cat Transfer | — | 84.3% | yeni | Yüksek kategori doğruluğu |
| Kategori sayısı | 9 | 6 | -3 | Daha az ama daha sağlam |
| Vocab | ~170 | 306 | +136 | 1.8x artış |
| Load+test süresi | — | ~1 sn | yeni | Hızlı Faz 6 rebuild |

### 2.2 15K 7-Domain — Eğitim Tamamlandı

**Tüm fazlar tamamlandı** (68 dakika eğitim + 0.86 sn test).

| Metrik | 10K | 15K v6.x | 15K v7.1 | 15K v7.2 | Yorum |
|--------|-----|----------|----------|----------|-------|
| Kategori sayısı | 6 | 22 | 22 | 22 | Stabil |
| Vocab | 306 | ~782 | ~782 | ~782 | Stabil |
| Test | 63.1% | 58.2% | 58.2% | **58.2%** | Sıfır regresyon |
| OOV | 46.4% | 39.7% | 39.7% | **39.6%** | ≈Aynı |
| QA Metrik 1 | 69.8% | 68.1% | 65.9% | **65.9%** | Stabil |
| SOV | — | 68.8% | 64.6% | **64.6%** | Stabil |
| Load+test süresi | ~1 sn | ~0.9 sn | 0.86 sn | **0.88 sn** | sent_store rebuild dahil |

### 2.3 v7.0/v7.1 SPEAK Kalite Değerlendirmesi

**v7.0 — Co-Occurrence Context Boost (Domain Bleeding Fix):**
- Önceki durum: Her soru "antrenör karga savunmacı" gibi D2(spor) kelimeleri üretiyordu
- Sonra: Domain-uyumlu cevaplar, domain bleeding %80+ azaldı

**v7.1 — Subject Injection + Position-Aware Protein Folding (SOV Fix):**
- Özne doğru ama nesne hâlâ yanlış ("kedi yarın yer alır")

**v7.2 — Object Word Injection + wpos Off-by-One Fix + sent_store Rebuild:**

| Soru | v6.x | v7.1 | v7.2 |
|------|------|------|------|
| kedi ne yer | antrenör karga savunmacı | kedi yarın yer alır | **kedi çorbayı yer** ✅ |
| futbolcu ne yapar | antrenör yapar savunuyor | futbolcu yapar savunuyor | **futbolcu blok çalışıyor** ✅ |
| anne ne pişirir | antrenör yapıyor | anne masalı sever | **anne çayı kullanır** ✅ |
| kartal ne yer | karga kaplan yargılıyor | kartal yaprakları sever | **kartal eti yer** 🔥 |
| doktor ne yapar | antrenör yapar | doktor yapar | **doktor koşu ediyor** ⚠️ |
| köpek ne içer | antrenör kalkıyor | — | **köpek üzümü içer** ✅ |
| ayı balı ne yer | ay bir yağıyor | — | **ayı patatesi sever** ✅ |
| yağmur ne yapar | antrenör hava | rüzgâr hava hazırlıyor | **yağmur skor ıslatır** ⚠️ |

**Kazanımlar:**
- ✅ Özne %100 doğru (subject injection)
- ✅ Nesne %62.5 doğru domain (8/8 → 5/8 mükemmel, 2/8 cross-domain leak)
- ✅ SOV yapısı %100 doğru (özne-nesne-fiil sırası)
- ✅ Domain bleeding tamamen giderildi
- ✅ Batch metriklerde sıfır regresyon

**Kalan sorunlar (v7.3+ için):**
- ⚠️ Cross-domain co-occurrence leak: "doktor→koşu", "yağmur→skor"
  Neden: category filter kaldırıldı (CAT_3 için gerekliydi), bazı domain'ler karışıyor
  Çözüm: domain-ID bilgisi veya eğitim verisinde domain izolasyonu
- ⚠️ Fiil seçimi: "kullanır", "çalışıyor" bazen domain-dışı
- ❌ Absurd sorulara hâlâ confident cevap veriyor

**v7.2 Stress Test:** Batch metrikleri v7.1 ile birebir aynı (sıfır regresyon)

### 2.3 İlk Kategori Haritası — Dış Gözlemci Analizi (İki Ayrı İnceleme)

**22 kategorinin ilk okuması:**

```
BÜYÜK KATEGORİLER (yapısal roller):
  CAT_0  (0.85) [237+ kelime] — MEGA ÖZNE: tüm domain'lerin özneleri
         kaleci, tavuk, doçent, şimdi... ⚠️ zaman+özne karışması
  CAT_1  (0.96) [242+ kelime] — TÜM FİİLLER: domain-bağımsız fiil kümesi
         zıplıyor, bulur, hesaplıyor, sürer...
  CAT_3  (0.68) [130+ kelime] — TÜM NESNELER: akuzatif formlar
         ağırlığı, tohumu, molekülü, yolcuyu...

DOMAIN-SPESİFİK KÜMELER (gerçek ayrışma):
  CAT_2  (0.63) — Spor mekanları: kortta, pistinde, havuzda
  CAT_5  (0.48) — Doğa nesneleri: ağaçları, kıyıyı, toprağı ← D4!
  CAT_8  (0.89) — Ulaşım mekanları: durrakta, köprüde, istasyonda ← D6!
  CAT_9  (0.60) — Doğa mekanları: deltada, kutuplarda, bozkırda ← D4!
  CAT_10 (0.79) — Bilim nesneleri: sonucu, problemi, hipotezi ← D5!
  CAT_12 (0.68) — Bilim mekanları: observatoryada, laboratuvarda ← D5!
  CAT_14 (0.83) — Spor terimleri: blok, antrenman, skor ← D2!

GRAMER KATEGORİLERİ:
  CAT_4  (0.78) — Soru kelimeleri: yapar, zaman, yapıyor, renk
  CAT_6  (0.44) — Genel mekanlar: okulda, evde, bahçede
  CAT_7  (0.60) — Zarflar: hızlıca, sessizce, dikkatle
  CAT_11 (0.85) — Bileşik fiil öğeleri: pas, inşa, park, tamir, analiz

KÜÇÜK/ŞÜPHELI KATEGORİLER:
  CAT_13 (0.66) — buz, ip (noise veya proto-category?)
  CAT_15 (0.43) — nazikçe, salonda (noise?)
  CAT_17 (0.97) — hekimi, sahibi, insanı (bileşik isimler)
  CAT_18 (1.00) — odasında, sofrada, markette (aile mekanları D3)
  CAT_19 (0.47) — köyde, plajda
  CAT_20 (0.76) — tropiklerde, vadide
  CAT_21 (1.00) — su, ankara, istanbul (bilgi entity'leri)
```

**Sentez — İki gözlemcinin ortak tespitleri:**

✅ **Kazanımlar (kesin):**
- Domain ayrışması VAR — D2, D4, D5, D6 ayrı kümeler oluşturmuş
- Gramer rolleri keşfedilmiş — fiil, nesne, zarf, mekan ayrı kategoriler
- Domain-içi alt yapı — D4'te nesne (CAT_5) vs mekan (CAT_9) ayrımı (önemli!)
- Cosine similarity 7 domain'de çalışıyor — GA şimdilik gerekmiyor

⚠️ **Uyarılar (kritik):**
- **CAT_0 mega-cluster:** 237+ kelime tek kategoride → özne + zaman karışması.
  Tüm domain özneleri benzer pozisyonda (cümle başı) geçiyor, esim() bunları
  birleştiriyor. Bu "representation limit" sinyali olabilir.
- **CAT_1 fiil birleşimi:** Tüm fiiller tek kategori → iki olasılık:
  - (A) İYİ: sistem "fiil" kavramını keşfetti
  - (B) KÖTÜ: fiiller arası anlam ayrımı yapamıyor
  - TEST: "doktor top atar" vs "futbolcu ameliyat yapar" — doğru genelleme mi?
- **Syntactic vs Semantic:** Şu an görülen = syntactic clustering (rol bazlı).
  AGI için gereken = semantic clustering (anlam bazlı). Bu sınır Aşama 2-3 ile aşılacak.

❌ **Henüz yok:**
- Ontoloji (canlı/cansız ayrımı)
- Nedensellik
- Fiiller arası anlam farkı

**Küçük kategoriler (CAT_13, CAT_15, CAT_19, CAT_20):**
İki ihtimal: (A) Noise — rastgele bölünme, (B) Proto-category — henüz büyümemiş
gerçek kategori. Test: yeni veri eklediğinde büyüyor mu? Büyürse gerçek, yok olursa noise.

**En kritik test sorusu:** "Bu sistem kelime türü mü öğreniyor, yoksa anlam mı?"
Şu anki cevap: kelime türü + domain. Anlam henüz yok. Ama bu beklenen —
Aşama 1'in (dil kalıpları) doğal sınırı.

---

## 3. Mimari Kararlar (Yeni)

### 3.1 Hızlı Faz 6 Rebuild vs Serialization
prompt_prior_qa matrisini serialize etmek yerine, load sonrası train verisinden rebuild:
- **Pro:** Model format değişmiyor (RIBO_VERSION=3 korunuyor), geriye uyumluluk
- **Pro:** 0.01 saniye süre — ihmal edilebilir
- **Pro:** Unity Build sırası sorunu yok
- **Con:** Train verisi dosyası gerekiyor (sadece model dosyası yetmiyor)
- **Karar:** Pragmatik, çalışıyor, ileride format v4'te serialize edilebilir

### 3.2 Tekrarlı Öğrenme İdempotency
ribozom_correct() içinde Lizozom lookup:
- Aynı soru-cevap → sadece hit_count, ağır eğitim atla
- Farklı cevap → tam eğitim yolu (güncelleme)
- **Etki:** Model şişmesi sıfıra indi, kullanıcı deneyimi aynı

### 3.3 v7.0 Co-Occurrence Context Boost (Mimari Karar)
SPEAK domain bleeding'in kök nedeni: Protein Folding tüm kategorilerden eşit ağırlıkta
aday çekiyor, q_dom=0 (hayvan) olsa bile CAT_14(spor) kelimeleri eşit şansla geliyor.

**Çözüm:** build_speak_cooc_boost() — sorundaki icerik kelimelerini Golgi sent_store'da
arayarak co-occurring kelime tablosu çıkar. Protein Folding bu tabloyu kullanarak
domain-dışı kelimeleri baskıla.

**Mekanizma zinciri:**
1. ISF (Inverse Sentence Frequency): Nadir sorgu kelimelerine yüksek ağırlık
2. Stop-word cutoff: >200 cümlede geçen kelimeler (yapar) ISF=0
3. Category filter: Sadece beklenen cevap kategorisindeki kelimeler sayılır
4. IDF on co-occurring words: Nadir co-occurring kelimelere bonus
5. Position-aware char boost: wpos[] ile SOV slot'una göre karakter boost'u
6. Non-co-occurring penalty: Domain-dışı kelimeler 0.05x ağırlık

**Yan etki:** QA -2.2%, SOV -4.2% — char-level boost bazen doğru trie tahminini bozuyor.
Subject injection'ın etkisi SIFIR (A/B test ile kanıtlandı).

### 3.4 v7.1 Subject Word Injection (Mimari Karar)
Char-level boost ilk karakteri bile değiştiremiyor (trie çok güçlü).
Çözüm: Karakter tahmini yerine, qmem'deki en yüksek subject-ratio kelimeyi doğrudan
SPEAK buffer'a yaz. predict_meta_v5 ile state machine'i güncelle.

**CAT_0 mega-cluster kararı:** Üç bağımsız analiz (kullanıcı, asistan, dış gözlemci)
"CAT_0'ı bölme, decoder'ı düzelt" üzerinde birleşti. CAT_0'ın 237 özneyi tek kategoride
toplaması bir "representation limit" değil, "abstraction miracle" — tüm özneler benzer
pozisyonda geçiyor ve sistem bunu keşfetmiş. Sorun kategorizasyonda değil, SPEAK
decode aşamasında bağlam kullanılmamasıydı.

### 3.5 wpos Rebuild (Mimari Karar)
wpos[] (pozisyonel imza) ribo31.bin'e serialize edilmiyor. Load sonrası tüm wpos sıfır →
subject injection ve position-aware boost çalışmıyor. Çözüm: Hızlı Faz 6'ya wpos rebuild
eklendi. Train verisinden content_pos_bucket() ile her kelimenin SOV pozisyonunu yeniden hesapla.

### 3.6 v7.2 Object Word Injection + Kritik Bug Fix'ler (Mimari Karar)

**5 bug tek seferde düzeltildi:**

1. **wpos off-by-one** (birincil bug): `g_speak_word_count` inject çağrılarından SONRA
   artırılıyordu. Protein Folding nesne slotunda `exp_bucket=0` (özne) kullanıyordu,
   `exp_bucket=3` (nesne) yerine. Fix: `eff_wc = g_speak_word_count + 1`.

2. **sent_store boş** (load-only mod): golgi_store() persist edilmiyor, Hızlı Faz 6'da
   rebuild edilmiyordu → n_sent=0 → build_speak_cooc_boost() erken dönüyordu.
   Fix: Faz 6'ya sent_store rebuild eklendi (8657 cümle, <0.01sn).

3. **ISF cutoff sabit 200**: 15K corpus'ta "kedi" 300+ cümlede geçiyor → ISF=0 →
   co-occurrence hiç hesaplanmıyordu. Fix: cutoff = n_sent/20 (corpus-orantılı).

4. **Category filter CAT_3'ü engelliyor**: prompt_prior_qa sadece ilk kelime geçişlerini
   saklıyor → CAT_3 (nesneler) hiçbir zaman expected_cat'te değildi → tüm nesne
   kelimeleri cooc_skip'e düşüyordu. Fix: category filter kaldırıldı, wpos bucket-3
   filtresi nesne/non-nesne ayrımını daha doğru yapıyor.

5. **Copy Reflex fiili nesne slotuna kopyalıyor**: "kedi ne yer" → qmem'deki "yer"
   nesne slotunda boost ediliyordu. Fix: wpos bucket filtresi eklendi — nesne slotunda
   sadece bucket 3 ≥%20 olan kelimeler kopyalanır.

**Object Word Injection:** Subject injection'ın aynısı ama nesne slotu için.
co-occurrence skoru × wpos bucket-3 oranı ile en iyi nesne kelimesini seç,
doğrudan buffer'a yaz. Karakter-seviye bias'ın trie'yi yenememesi sorununu
tamamen bypass eder.

**Sonuç:** "antrenör karga savunmacı" → "kartal eti yer" (tek günde).

### 3.7 7-Domain Genişleme Stratejisi
- D0 (Hayvan-Yemek) %37 ile en büyük domain — baseline koruması
- Her yeni domain ~1300 cümle — kategori keşfi için minimum yeterli
- D4 (Doğa) SV cümleleri — SOV varsayımını zorlayan kasıtlı stres testi
- Cross-domain köprüler — kategori transferi ve genelleme için kritik

---

## 4. Çözülen Sorunlar (Bu Oturumda)

| Sorun | Çözüm | Etki |
|-------|-------|------|
| QA SPEAK çöp (load-only mod) | Hızlı Faz 6 rebuild | QA 49%→69.8% |
| Tekrarlı öğrenme şişmesi | Lizozom idempotency | Sıfır şişme |
| "hayır" algılanmıyor | Encoding-agnostic normalize | Tüm kodlamalar çalışıyor |
| Virgül yapışması | Skip comma+space after first word | Temiz cevap ayrıştırma |
| QA test dosyası ASCII/UTF-8 uyumsuzluğu | test_data'dan grep ile regenerate | Sahte düşüş giderildi |
| Lizozom persist derleme hatası | Tipler config.h'ye taşındı | Unity Build uyumlu |
| **v7.0** SPEAK domain bleeding | Co-occurrence context boost (ISF+IDF+cat filter) | "antrenör karga" → domain-uyumlu cevap |
| **v7.0** \x01/\x02 marker kirlenmesi | Content filter (skip markers, punctuation, single chars) | Co-occurrence temiz |
| **v7.0** "yapar" flooding | ISF=1/count + stop-word cutoff @200 | Nadir kelimelere yüksek ağırlık |
| **v7.0** Noisy location co-occurrence | Category filter via prompt_prior_qa | Sadece beklenen kategoriler |
| **v7.1** Yanlış ilk kelime (özne) | Subject word injection (qmem→buffer) | Özne %100 doğru |
| **v7.1** wpos load sonrası sıfır | wpos rebuild Hızlı Faz 6'ya eklendi | Position-aware boost aktif |
| **v7.1** COOC_K_BASE aşırı (15→12) | Deneysel ayar | SPEAK boş çıktı sorunu giderildi |
| **v7.2** wpos off-by-one (exp_bucket yanlış) | eff_wc = g_speak_word_count + 1 | Nesne slotunda doğru pozisyon filtresi |
| **v7.2** sent_store load sonrası boş | Faz 6'ya golgi_store rebuild | Co-occurrence 0→8657 cümle |
| **v7.2** ISF cutoff sabit 200 | n_sent/20 (corpus-orantılı) | "kedi" artık co-occurrence'ta |
| **v7.2** Category filter CAT_3 engelliyor | Filter kaldırıldı, wpos bucket filtresi | Nesne kelimeleri co-occurrence'ta |
| **v7.2** Copy Reflex fiili nesne slotuna | wpos bucket ≥%20 filtresi | "yer" artık nesne slotunda değil |
| **v7.2** Object Word Injection | cooc × bucket-3 ile doğrudan enjeksiyon | "kedi çorbayı yer" 🔥 |

---

## 5. Bilinen Eksiklikler ve Yol Haritası (Güncellendi)

### Tamamlandı ✅
1. ~~10K UTF-8 Türkçe veri~~
2. ~~Load-only mod (--train flag)~~
3. ~~QA SPEAK load sonrası çalışma~~
4. ~~Tekrarlı öğrenme idempotency~~
5. ~~Stress test altyapısı~~
6. ~~15K 7-domain veri jeneratörü~~

### Devam Eden 🔄
7. ~~15K eğitim~~ ✅ Tamamlandı (68 dk)
8. ~~Domain genelleme testi~~ ✅ Tamamlandı — 22 kategori, 7 domain ayrışması görüldü
9. ~~v7.0 Domain bleeding fix~~ ✅ Co-occurrence context boost
10. ~~v7.1 SOV word order fix~~ ✅ Subject injection + position-aware Protein Folding
11. ~~v7.2 Nesne/fiil seçimi~~ ✅ Object Word Injection + 5 kritik bug fix ("kartal eti yer")
12. **Aşama 2 — Koşullu Cümleler** ⛔ KAPATILDI (14 Nisan): representation kazanıldı, kullanım %0, QA −11 regresyon → rollback, Aşama 3 Sitoiskelet'e devredildi (bkz. Bölüm 8.8)

### Kısa Vade ⏳
12. **"Bilmiyorum" eşiği** — ribo_conf() düşükse "bilmiyorum" de
13. **Online cluster** — Yeni kelimeler runtime'da kategorilere eklenebilmeli
14. **Lizozom TF-IDF** — Bag-of-words Jaccard yerine kelime önem ağırlığı (T8 miss fix)
15. **test_qa_offd.txt UTF-8** — Off-diagonal test dosyası hâlâ eski ASCII
16. **Fiil tekrarı** — CAT_1 mega-cluster: domain-spesifik fiil seçimi zayıf ("yapar" her yerde)
17. **Nesne doğruluğu** — SPEAK 2. kelime bazen yanlış domain'den geliyor

### Orta Vade 🔜
13. Negatif pekiştirme (yanlış şablona ceza)
14. wctx pencere genişletme (2-kelime bağlam)
15. Morfoloji iyileştirmesi (BPE-tarzı alt-kelime)

### Uzun Vade (AGI Yönü) 🔮
16. Olumsuzluk/yokluk kavramı ("değil", "yok")
17. Nedensellik ("çünkü", "bu yüzden")
18. Coreference ("O kim?" çözümlemesi)
19. Hiyerarşik bellek (kelime→cümle→paragraf)

### AGI Aşama Haritası — Dilden Anlama Geçiş

**Temel tespit:** "Konuşmayı mükemmel öğrenmek" dil akıcılığı verir ama zeka
oluşturmaz. Bu, bugünkü büyük dil modellerinin düştüğü tuzaktır. Ribozom'un farkı:
sıfırdan inşa edildiği için her katmanı bilinçli eklenebilir.

```
┌─────────────────────────────────────────────────────────────┐
│  AŞAMA 1 — DİL (Pattern-Learning Infant)      [MEVCUT]     │
│  ✅ Kelime eşleştirme, kalıp öğrenme, basit genelleme       │
│  ✅ SOV yapısı, kategori keşfi, canlı düzeltme              │
│  🔄 7 domain genişleme (15K eğitim devam ediyor)            │
│  Hedef: Multi-domain SOV genellemesi + SV yapısal keşif     │
├─────────────────────────────────────────────────────────────┤
│  AŞAMA 2 — KOŞULLU BİLGİ (Dünya Modeli Tohumu)  [SIRADA]  │
│  "kedi açsa balık yer"  /  "kedi tokken balık yemez"        │
│  "yağmur yağarsa toprak ıslanır"                            │
│                                                             │
│  Mimari etki: DÜŞÜK — veri eklentisi yeterli                │
│    BOS → CAT_koşul → CAT_özne → CAT_nesne → CAT_fiil      │
│    "açsa", "tokken", "ıslanırsa" → yeni kategori (CAT_KOŞ) │
│    Mevcut mikrotübül geçiş matrisi bunu zaten destekliyor   │
│                                                             │
│  ⚡ Kritik tweak: Yüzey varyasyonları ekle                  │
│    "kedi açsa balık yer"         (koşul-özne-nesne-fiil)    │
│    "kedi aç olursa balık yer"    (analitik form)            │
│    "aç olan kedi balık yer"      (sıfat-isim formu)         │
│    "kedi açken balık yer"        (zarf-fiil formu)          │
│    → Aynı anlam, farklı yüzey = gerçek abstraction testi   │
│                                                             │
│  ❗ NEGATION (kritik ek — zeka = fark üretme):              │
│    "kedi aç değilse balık yemez"                            │
│    "yağmur yağmazsa yer ıslanmaz"                           │
│    → Negation olmadan sistem hep "pozitif ezber" yapar      │
│                                                             │
│  Kazanım: DURUM, KOŞUL, BAĞLAM + OLUMSUZLUK kavramları     │
├─────────────────────────────────────────────────────────────┤
│  AŞAMA 3 — NEDENSELLİK ZİNCİRİ                  [PLANLAMA]│
│  "yağmur → ıslak → kaygan" zinciri                         │
│  "çünkü" / "bu yüzden" → CAT_BAĞLAÇ kategorisi             │
│                                                             │
│  Mimari etki: ORTA — Vesicle Memory gerekli                 │
│    Her cümle bitiminde aktif bağlam bir vesicle'a paketlenir│
│    "Çünkü" geldiğinde önceki vesicle'a referans kurulur     │
│    CÜMLE_1.EOS → CAT_bağlaç → CÜMLE_2.BOS → ...            │
│                                                             │
│  ❗ Forward + Backward (kritik ek):                         │
│    Forward:  "yağmur yağarsa toprak ıslanır"                │
│    Forward:  "toprak ıslanırsa kaygan olur"                 │
│    Backward: "toprak neden ıslak? çünkü yağmur yağdı"      │
│    → İki yönlü zincir = gerçek akıl yürütme temeli         │
│    → Backward query olmadan sistem sadece "ileri tahmin"    │
│      yapar, "neden?" sorusuna cevap veremez                 │
│                                                             │
│  Kazanım: AGI'nin ilk gerçek çekirdeği — sebep-sonuç       │
├─────────────────────────────────────────────────────────────┤
│  AŞAMA 4 — ONTOLOJİ (Varlık Türleri)            [UZUN VAD]│
│  kedi (canlı, memeli) vs taş (cansız, nesne)               │
│  yağmur (süreç) vs dağ (kalıcı)                            │
│                                                             │
│  Mimari etki: YÜKSEK — özellik vektörü veya GA gerekebilir │
│    Seçenek A: Explicit özellikler (is_alive, is_process)    │
│    Seçenek B: Geometric Algebra (bivector ilişki temsili)   │
│    Karar tetikleyici: 15K'da cosine similarity yeterliliği  │
│                                                             │
│  Kazanım: "Anlam" — varlıkların doğası hakkında bilgi      │
├─────────────────────────────────────────────────────────────┤
│  AŞAMA 5 — ÜST-BİLİŞ (Meta-Cognition)          [VİZYON]  │
│  "Bilmiyorum" (güven eşiği)                                │
│  "Bunu daha önce yanılmıştım" (hata hafızası)              │
│  "Bu soru öncekiyle çelişiyor" (tutarlılık kontrolü)        │
│                                                             │
│  Kazanım: Sistem kendi bilgisini sorguluyor — AGI eşiği    │
└─────────────────────────────────────────────────────────────┘
```

**Kritik sıralama prensibi:** Her aşama bir öncekinin verisine ve altyapısına dayanır.
Aşama 2 (koşullu bilgi) Aşama 1 (SOV genellemesi) olmadan anlamsız — "açsa" koşulunu
öğretmeden önce "kedi balık yer" kalıbını sağlam bilmesi gerekiyor. Aşama 3 (nedensellik)
Aşama 2 olmadan imkansız — nedensellik zincirinin her halkası bir koşullu cümle.

**Aşama 2 tetikleyici:** 15K eğitim tamamlanıp sonuçlar ölçüldükten sonra.
Koşullu cümleler mevcut gen_15k_tr.py'ye eklenecek, mimari değişiklik gerektirmiyor.

---

## 6. Dosya Yapısı (13 Nisan 2026 — Güncel)

```
v3/files/
├── ribozom_main.c           — Ana dosya + interactive mode (~900+ satır)
├── ribozom_config.h          — Sabitler + Lizozom tipleri + dyn_norm
├── ribozom_core.h            — Şablonlar + LCRS Trie + rtrain/rpredict
├── ribozom_vocab.h           — ER + hash + esim + cluster + build_soft_cats
├── ribozom_golgi.h           — Golgi + cat templates + predict_meta
├── ribozom_micro.h           — Mikrotübül + morfoloji + incremental updates
├── ribozom_persist.h         — save/load model (ribo31.bin, RIBO_VERSION=3)
├── ribozom_qa.h              — QA state machine + SPEAK + Lizozom + correct()
│
├── gen_15k_tr.py             — ★ 15K 7-domain Türkçe veri jeneratörü
├── gen_10k_tr.py             — (eski) 10K tek domain jeneratör
├── train_data_15k.txt        — ★ 13600 satır eğitim verisi (7 domain, UTF-8)
├── test_data_15k.txt         — ★ 500 satır test verisi (7 domain)
├── oov_test_15k.txt          — ★ 100 satır OOV test verisi (3 domain)
├── train_data_10k.txt        — (eski) 9500 satır eğitim
├── test_data_10k.txt         — (eski) 500 satır test
├── oov_test_10k.txt          — (eski) 100 satır OOV
├── test_qa_v5.txt            — QA test çiftleri (test_data'dan türetilmiş)
├── stress_test.sh            — ★ Lizozom stress test scripti (10 senaryo)
│
├── ribo31.bin                — Serialize edilmiş model
├── MANIFESTO.md              — Proje vizyonu ve mimari
├── PROJE_DURUMU_v10K.md      — Önceki durum raporu
├── PROJE_DURUMU_v15K.md      — ★ BU DOKÜMAN
└── TEKNIK_BORC_v7.md         — Teknik borç kaydı
```

---

## 7. 15K Kırılma Sinyalleri — Ne Ölçülecek, Nasıl Yorumlanacak

> *Sadece accuracy'ye bakmak yanıltır. Asıl sinyal yapısal davranıştadır.*

### 🎯 Sinyal 1: Kategori Ayrışması (EN KRİTİK)

**Soru:** 7 domain gerçekten ayrışıyor mu?
- "doktor", "futbolcu", "anne" → aynı kategoriye mi düşüyor, yoksa ayrı cluster mı?
- **Beklenen:** 6 → 8-12 kategori

| Sonuç | Yorum | Aksiyon |
|-------|-------|---------|
| 8-12 kategori | ✅ Domain'ler ayrışıyor, genelleme başlıyor | Aşama 2'ye geç |
| 6 civarı kalır | ❌ Sistem genelleme yapmıyor, ezberliyor | esim() temsil gücünü sorgula |
| 15+ kategori | ⚠️ Aşırı parçalanma, over-clustering | Cluster threshold'u gözden geçir |

**DİKKAT:** Ayrışma ≠ ontoloji öğrendi. Kategoriler veri dağılımı sonucu da oluşabilir.
"yağmur" ayrı kategoriye düşse bile, sistem "cansız" kavramını anlamıyor — sadece "yağmur"
kelimesinin bağlamının diğerlerinden farklı olduğunu tespit ediyor. Gerçek ontoloji Aşama 4.

**Stabilite kontrolü:** Kategori sayısı stabil mi? Aynı veri farklı seed/sırayla
çalıştırıldığında sürekli değişiyorsa → clustering unstable, sayıya güvenme.

### 🎯 Sinyal 2: Cross-Domain Davranış

**Soru:** Köprü cümleleri ("anne kediyi sever") mantıklı genelleniyor mu?
- Sistem D3(aile) öznesini D0(hayvan) nesnesiyle doğru birleştiriyor mu?
- Yoksa saçma kategoriye mi atıyor?

| Sonuç | Yorum |
|-------|-------|
| Doğru genelleme | ✅ Abstraction başlamış — kategori geçişleri domain-bağımsız |
| Saçma atama | ❌ Kategoriler domain'e kilitli — transfer öğrenme yok |

### 🎯 Sinyal 3: SV Etkisi (Yapısal Keşif)

**Soru:** "yağmur yağar" (SV) kalıbı SOV'dan ayrı öğreniliyor mu?
- D4'teki ~1064 nesnesiz cümle geçiş matrisinde nasıl görünüyor?
- BOS → CAT_doğa → EOS (nesne atlayarak) mı, yoksa SOV'a zorla sığdırma mı?

| Sonuç | Yorum |
|-------|-------|
| Ayrı geçiş kalıbı | 🔥 Büyük başarı — sistem cümle yapısını keşfediyor |
| SOV'a zorla sığdırma | ❌ "Sentence structure blind" — yapısal esneklik yok |
| Karma/karmaşık | ⚠️ Sinyal belirsiz — daha fazla SV verisi gerekebilir |

**Gizli genelleme testi:** Train'de "yağmur yağar", "rüzgâr eser" var. Test'te
"güneş parlar" — eğer doğru genellerse SV öğrenilmiş, saçmalarsa ezber.

### 🎯 Sinyal 4: OOV Davranışı (Gerçek Öğrenme Testi)

**Soru:** Yeni domain'lerden OOV kelimelerde ne yapıyor?
- D1 OOV ("psikiyatrist") → en yakın meslek kategorisini mi seçiyor?
- D4 OOV ("tsunami") → doğa kategorisine mi düşüyor?

| Sonuç | Yorum |
|-------|-------|
| Mantıklı tahmin | ✅ Gerçek genelleme — bağlam transferi çalışıyor |
| Rastgele/varsayılan | ❌ OOV hâlâ kör — distributional signal yetersiz |

**Overgeneralization testi:** "aslan top oynar mı?" — sistem çok emin cevap veriyorsa
overgeneralization var. İdeal: düşük confidence veya kararsızlık.

### 🎯 Meta-Sinyal: HATALARA ODAKLAN

> *Sistemin gerçek yapısı metriklerde değil, hatalarda ortaya çıkar.*
> *Sonuç geldiğinde sakın "genel olarak iyi görünüyor" deme.*
> *Sor: "Bu sistem nerede aptal?"*

15K sonucu geldiğinde en önce bakılacaklar:
1. **En saçma cevaplar** — SPEAK'in ürettiği en anlamsız cümleler neler?
2. **En beklenmedik hatalar** — doğru bildiği bir şeyi neden yanlış yaptı?
3. **Domain karışması** — hangi domain çiftleri birbirine en çok karışıyor?
4. **Confidence dağılımı** — yanlış cevaplarda confidence yüksek mi? (tehlikeli!)

### 🛡️ Yorum Guardrail'leri (False Positive/Negative Koruması)

#### Guardrail #1 — Tek metrikle karar verme
Kategori sayısı 10 ama OOV kötü ve SV başarısız → bu **false success**.
**Kural:** En az 2 bağımsız sinyal aynı şeyi söylüyorsa karar ver.

#### Guardrail #2 — En kötü 5 örnek zorunlu raporu
Her testten sonra **zorunlu** listeleme:
- En kötü 5 cevap
- En saçma 5 cevap
- En yüksek confidence'lı yanlış 5 cevap

Sistem kendini ortalamada değil, **uçlarda** ele verir.

#### Guardrail #3 — Sessiz başarısızlık tespiti
Model yanlış ama makul cevap verir, dikkat çekmez. Test:
- "kedi uçar mı?" → confident cevap veriyorsa ❌ gerçeklik filtresi yok
- "taş konuşur mu?" → confident cevap veriyorsa ❌
- Bu sorularda beklenen: düşük confidence veya "bilmiyorum"

### 🧪 Impossible/Absurd Test Seti (Gerçeklik Sınırı Ölçümü)

15K sonrası interaktif modda çalıştırılacak absurd sorular:
```
taş ne yer ?            → beklenen: düşük confidence / saçma cevap
güneş ne içer ?         → beklenen: düşük confidence / saçma cevap
araba ne düşünür ?      → beklenen: düşük confidence / saçma cevap
kedi ne uçar ?          → beklenen: kararsızlık
dağ ne pişirir ?        → beklenen: düşük confidence / saçma cevap
```
**Değerlendirme:** Eğer sistem bu sorulara confident cevap veriyorsa,
gerçeklik filtresi YOK demektir — "bilmiyorum" eşiği acil öncelik olur.

### Ek Açık Sorular

5. **Vocab ölçekleme:** 782 kelime ile hash çakışma oranı, cluster() süresi,
   esim() kalitesi nasıl değişiyor?

6. **D6 dual-role:** "otobüs yolcuyu taşır" (özne) vs "şoför otobüsü sürer"
   (nesne) — aynı kelime iki farklı konumda. Kategori ataması tutarlı mı?

7. **Negation + Contradiction testi (Aşama 2 hazırlığı):**
   "kedi balık yer" + "kedi balık yemez" → sistem ne yapar?
   İdeal: conflict algıla, confidence düşür, veya "bilmiyorum" de.
   Bu test şu an yapılamaz (negation verisi yok) ama Aşama 2'nin başarı ölçütü olacak.

8. **Temporal contradiction (Aşama 2-3 köprüsü):**
   Interaktif modda sırayla öğret:
   - "kedi balık yer" (düzeltme ile öğret)
   - "kedi artık balık yemez" (yeni düzeltme)
   - "kedi ne yer?" (soru sor)
   → Hangisi geçerli? Sistem memory update + conflict resolution yapabiliyor mu?
   → Lizozom son kaydı alır (mekanik çözüm), ama "artık" zamansal operatörünü
     anlaması gerçek zeka sıçraması olur.

---

## 8. Dış İnceleme ve Yanıtlar

*İncelemeler v10K raporunda detaylı şekilde belgelenmiştir (Bölüm 7-8). Özet:*

### Kabul Edilen ve Uygulanan
- ✅ Statik ER → Online cluster (yol haritasında, sıra 10)
- ✅ "Bilmiyorum" eşiği → ribo_conf() altyapısı mevcut (sıra 9)
- ✅ Tekrar-tabanlı öğrenme → İdempotency fix ile çözüldü

### Kabul Edilen Ama Ertelenen
- ⏳ Long-range dependencies → Mikrotübül kısmen çözüyor, hiyerarşik bellek ileride
- ⏳ Olumsuzluk/nedensellik → AGI yol haritasında
- ⏳ Global state → Algoritmalar stabilize olduktan sonra refactor

### Reddedilen veya Nüanslanan
- "Vektör uzayı şart" → esim() zaten distributional semantics yapıyor
- "Trie ile AGI mümkün değil" → Trie tek organel, sistem 6+ bileşenden oluşuyor

---

*"Dar domain'de başarı illüzyondur. Gerçek test, bilinmeyen bir dünyada ilk adımı atabilmektir."*

*— 15K 7-domain genişleme bu testi karşılıyor.*

---

## 9. Creator Notes

> *"Basitlik, karmaşıklığın en üst noktasıdır."* — Leonardo da Vinci

### 9.1 Geometric Algebra (GA) — Yedek Plan

Şu an GA için erken. Neden:

Ribozom'un mevcut gücü basitliğinde. `esim()` cosine similarity, 10 satır C kodu,
anında çalışıyor. 7 domain'de 6 kategoriden 12'ye çıkabilecek mi diye test ediyoruz.
Bu temel sorunun cevabını bilmeden üzerine GA eklemek, temeli görmeden çatı inşa
etmek olur.

**GA entegrasyonu ciddi bir mimari değişiklik:**
- wctx dizilerini multivector struct'lara dönüştürmek = ribozom_vocab.h'ın yarısını yeniden yazmak
- Bellek: G4 multivector = 16 float bileşen, mevcut wctx = MAX_CTX int
- Boyut ve tip değişimi tüm downstream fonksiyonları etkiler

**Ama fikri çöpe atma.** Şu senaryoda devreye girer:

Eğer 15K 7-domain testi sonucunda kategori keşfi başarısız olursa — yani cosine
similarity farklı domain'lerdeki yapısal benzerlikleri yakalayamazsa — o zaman
temsil gücü sorunu var demektir. Ve o noktada GA'nın bivector ilişki temsili
gerçek bir çözüm olur.

**GA karar ağacı:**
1. 15K 7-domain sonuçlarını gör
2. Kategori sayısı artıyor mu, cross-domain genelleme var mı?
3. Eğer cosine similarity yetersiz kalırsa → GA'yı `esim()` yerine prototiple
4. Eğer çalışıyorsa → ölçeklemeye devam et, GA'yı Aşama 4 (Ontoloji) için sakla

**Boyut laneti çözümü — GA'yı kategori uzayında çalıştır:**

$2^n$ bileşen sorunu GA'nın bilinen zayıflığı. 10.000 boyutlu kelime uzayında
full multivector hesaplamak imkansız ($2^{10000}$ bileşen). Ama Ribozom'un mimarisi
bunu doğal olarak çözüyor:

```
Kelime uzayı:     782 kelime  →  2^782 bileşen  →  İMKANSIZ
Kategori uzayı:   12 kategori →  2^12 = 4096     →  KOLAY
Grade-kısıtlı:    Grade 0,1,2 →  1+12+66 = 79    →  TRİVİAL (316 byte)
```

`cluster()` zaten boyut sıkıştırma yapıyor — yüzlerce kelimeyi düzinelerce kategoriye
indiriyor. GA bu sıkıştırılmış uzayda çalışır:

- **Grade 0 (scalar):** Benzerlik skoru — mevcut `esim()` karşılığı
- **Grade 1 (vector):** Kategori temsili — mevcut `cats[c]` karşılığı
- **Grade 2 (bivector):** Kategori ilişkileri — `CAT_özne ∧ CAT_fiil` = yönlü ilişki
  Mevcut `trans[c1][c2]` geçiş matrisi bunu sayısal tutuyor ama yönü kaybediyor.
  Bivector: `e_özne ∧ e_fiil ≠ e_fiil ∧ e_özne` — yön korunuyor.

12 kategori: 12×11/2 = 66 bivector = 66 float = **264 byte**. Toplam multivector 316 byte.
Milyar parametreli modeller için kabus olan şey, Ribozom için ihmal edilebilir.

Bu, neuroscience'taki **sparse coding** prensibi: beyin milyarlarca sinaptik girdiyi
birkaç yüz kavrama sıkıştırıyor, üst seviye akıl yürütme sıkıştırılmış uzayda yapılıyor.

```c
// Olası GA struct (kategoriler üzerinde)
typedef struct {
    float scalar;                              // grade 0: 1 bileşen
    float vec[MAX_CATS];                       // grade 1: n bileşen
    float bivec[MAX_CATS*(MAX_CATS-1)/2];      // grade 2: n(n-1)/2 bileşen
} CatMultivector;  // 12 kategori → 79 float = 316 byte
```

### 9.2 Ontoloji Yanılsaması Uyarısı

`esim()` ile ontoloji "kısmen çözülüyor" demek tehlikeli bir ifade. Gerçek durum:

- `esim()` **bağlam benzerliği** yakalar ✅ — "kedi" ve "köpek" benzer bağlamlarda geçer
- `esim()` **ontoloji vermez** ❌ — "kedi koşar" ile "araba gider" bağlam-benzer olabilir
  ama biri canlı, biri değil

Eğer 15K'da "yağmur" ayrı kategoriye düşerse, bu **ontoloji öğrendi demek değil**.
Sadece "yağmur" kelimesinin veri dağılımında diğerlerinden farklı konumlandığını
tespit etmiş. Gerçek ontoloji (canlı/cansız, süreç/nesne) Aşama 4'te açık temsil
gerektirir.

**Test yöntemi:** "kedi koşar" ve "araba gider" aynı kategoride mi? Eğer evetse,
sistem yapısal benzerliği yakalamış ama ontolojik farkı kaçırmış.

### 9.3 "Dilden Anlama" Geçişi Hakkında

Projenin kritik kavşağı şu: dil akıcılığı ≠ zeka. Büyük dil modellerinin tuzağına
düşmemek için, Aşama 1 (dil kalıpları) tamamlanır tamamlanmaz Aşama 2'ye (koşullu
bilgi) geçmek gerekiyor. Bu geçiş mimari değil, **veri** geçişi:

```
Aşama 1 verisi:  "kedi balık yer"                         (kalıp)

Aşama 2 verisi:  "kedi açsa balık yer"                     (koşul)
                  "kedi aç olursa balık yer"                (analitik form)
                  "aç olan kedi balık yer"                  (sıfat-isim)
                  "kedi açken balık yer"                    (zarf-fiil)
                  → Aynı anlam, farklı yüzey = abstraction testi

                  "kedi aç değilse balık yemez"            (NEGATION)
                  "yağmur yağmazsa yer ıslanmaz"           (NEGATION)
                  → Olumsuzluk olmadan sistem "pozitif ezber" tuzağına düşer

Aşama 3 verisi:  "yağmur yağarsa toprak ıslanır"          (forward chain)
                  "toprak ıslanırsa kaygan olur"            (forward chain)
                  "toprak neden ıslak? çünkü yağmur yağdı" (backward query)
                  → İki yönlü zincir = gerçek akıl yürütme
```

Mevcut mikrotübül geçiş matrisi Aşama 2'yi **mimari değişiklik olmadan** destekliyor.
"açsa" ve "tokken" yeni kelimeler olarak vocab'a girer, cluster() bunları koşullu
operatörler olarak gruplayabilir (benzer bağlamlarda geçtikleri için).

Aşama 3 ise Vesicle Memory gibi yeni bir organelle gerektiriyor — cümle-arası referans.
Bu noktada mimari sıçrama kaçınılmaz ama altyapı (Lizozom, Mikrotübül) temel sağlıyor.

> *"Doğru sıra: önce yürü, sonra koş, sonra uç. Ama yürürken uçuş planını çizmeye başla."*

---

## 8. Aşama 2 — Koşullu Cümleler (14 Nisan, Devam Eden)

### 8.1 Strateji Kararı
v7.2 tamamlandıktan sonra iki yol vardı: (a) cross-domain leak + fiil seçimi
optimizasyonu (%62.5 → ~%75 SPEAK kalitesi), (b) Aşama 2'ye atlayış.
Karar: **Aşama 2**. Gerekçe — SPEAK optimizasyonu diminishing returns, Aşama 2
yeni bir bilişsel boyut (koşul operatörü) ekliyor. Aynı efor, daha büyük sıçrama.

### 8.2 Veri
- `train_data_15k_pre_s2.txt` — v7.2 mühürlü baseline (13600 satır, 4943 QA pair)
- `conditional_train.txt` — 2000 koşullu cümle, 850 QA pair
- `train_data_15k.txt` — birleşik 15600 satır, 5793 QA pair (eğitim girdisi)
- Not: `ribozom_main.c` dosya adını hardcoded okuyor — `--train <path>` desteklenmiyor.
  Veri değişikliği için dosyayı yerinde değiştir + backup al protokolü kullanıldı.

### 8.3 Mini Sanity (1500 satır, 14 Nisan sabah)
5 dk hızlı test — full eğitime girmeden sinyal doğrulama.
Dataset: 500 koşullu + 1000 normal (shuf), 606 QA pair.

**Sonuç — Senaryo A (Başarı):**
```
CAT_14 (cohesion 0.94):  sorarsa, çalışırsa, yağarsa, çalarsa, ararsa, isterse, ...+26
CAT_21 (cohesion 1.00):  geçerse, düşerse, korkarsa, sevinirse
CAT_27 (cohesion 0.53):  pişer, demlenir, nedir, spor, ver, uyanır        (OUTCOME fiilleri)
```

**Gözlemler:**
- `-sa/-se` eki **ayrı kategori** oluşturdu, cohesion tavan (0.94 ve 1.00)
- **İki** koşul cluster'ı çıktı — muhtemelen aspect/tense ayrımı
  (alışkanlık-koşulu vs olay-koşulu hipotezi)
- CAT_27'de `-er/-ir` aorist outcome fiilleri toplandı → koşul→sonuç **ikili yapısı**
  representation seviyesinde belirmiş

**Kritik ayrım:** Bu sadece **representation** sinyali. Asıl test SPEAK'te
"condition usage" — sistem bu bilgiyi karar verirken kullanıyor mu?

### 8.4 Test Protokolü (Full Eğitim Sonrası)

**A. Cluster analizi:**
1. CAT_14 ve CAT_21 tam liste çıkar
2. **Altın test** — `çalışırsa` vs `çalışsaydı` aynı cluster'da mı?
   - Aynı → sistem sadece `-sa` ekine bakıyor
   - Farklı → sistem **aspect/tense** ayrıştırıyor

**B. Distribution probing (3-yönlü delta):**
```
P(w | "kedi ne yapar?")           → baseline
P(w | "kedi açsa ne yapar?")      → condition+
P(w | "kedi tokken ne yapar?")    → condition-
```
Değerlendirme:
- Üç dağılım da aynı → koşul **tamamen pasif** (Senaryo B)
- Koşullu ≠ baseline ama iki koşul birbirine benziyor → koşul "fark ediliyor" ama polarite okunmuyor
- İki koşul **zıt** → gerçek condition usage ✔

**C. Negation + condition:**
- `yağmur yağarsa` vs `yağmur yağmazsa` delta
- Zıt dağılım = polarite + koşul **birlikte** işliyor (çift büyük sıçrama)

**D. SPEAK direkt test:**
- `kar yağarsa ne olur?`
- `kedi açsa ne yapar?` vs `kedi tokken ne yapar?`
- `çocuk okula gitmezse ne olur?`
- `kedi aç değilse ne yapar?`

### 8.5 Teşhis Matrisi

| Cluster | Distribution | Teşhis | Aksiyon |
|---------|--------------|--------|---------|
| ✔ | ✔ | Gerçek condition learning | Aşama 2 zaferi, Aşama 3 planlama |
| ✔ | ❌ | Representation var, usage yok | Condition-weight injection (Ek-8) |
| ❌ | ❌ | Kapasite sınırı | Mimari genişletme gerekli |
| ❌ | ✔ | İmplicit öğrenme | Derinlemesine çalış, çok ilginç |

### 8.6 Açık Mimari Risk — WPOS Bucket Çakışması
Mevcut sistem varsayımı: `[özne] [nesne] [fiil]` — wpos bucket-4 son/ana fiil.
Koşullu yapı: `[özne] [koşul-fiil] [nesne] [ana-fiil]` — **iki fiil**.

Eğer koşul fiilleri (`açsa`, `yağarsa`) bucket-3 veya bucket-4'e düşüp ana fiille
çakışırsa Protein Folding patolojisi yaşanır. Full eğitim sonrası wpos dağılımı
inceleme zorunlu. Çözüm adayı (gerekirse): bucket genişletme (4→5) veya ayrı
"condition bucket" tanımı.

### 8.7 Full Eğitim Sonucu (14 Nisan, 12:20)
- Başlangıç: 14 Nisan sabah, 15600 satır (13600 baseline + 2000 koşullu)
- Toplam süre: **8760.10 sn = 146 dk (2 saat 26 dk)**
- Faz 1: 4575.69 sn (76 dk) — 25 kategori oluştu
- Faz 3: 4183.15 sn (70 dk) — şablon eğitimi
- Çıktı: `s2_full.log` (370 satır), `ribo31.bin` (Aşama 2 modeli)

### 8.8 Aşama 2 RESMİ KAPANIŞ (14 Nisan, 12:35)

#### 8.8.1 Metrik Regresyonu (sıfır-tolerans ihlali)

| Metrik | v7.2 Baseline | Aşama 2 (15600 satır) | Delta |
|---|---|---|---|
| QA | **76.2%** | 65.2% | **−11.0 puan** |
| OOV | **50.0%** | 39.2% | **−10.8 puan** |
| SOV iskelet | — | 63.8% | — |
| Off-diagonal | ~0 | 2.0% (3/150) | istatistik gürültü |

Regresyon büyüklüğü (≥10 puan) Prensip 0.7 (*sıfır-tolerans*) ihlali.

#### 8.8.2 Üç Teşhis Ölçümü (kapanış kanıtları)

**Ölçüm 1 — CAT_21 suffix_homogeneity (kirli cluster):**
- 36 üye, 28 ayrık son-ek tipi
- Normalize homogeneity = `log2(28)/log2(36) = 4.81/5.17 = 0.93` (çok yüksek çeşitlilik)
- Ek aileleri karışık: `-mi/-imi/-kimi` (nesne-hal), `-bi/-ibi/-hibi` (nesne-hal),
  `-ı/-nı` (nesne-hal), `-sa/-rsa/-arsa` (koşul)
- Mikrotubul kararsızlığı: `CAT_21 → CAT_4 = %28 (391/1383)`, kıyasla
  temiz kümeler `%100` (CAT_13→CAT_1, CAT_17→CAT_4)
- **Kıyas — temiz küçük koşul kümesi CAT_18**: 2 üye, ek dağılımı
  `-sa(2)+-rsa(1)+-arsa(1) = 4/6` koşul-ailesi. Sinyal var ama kütle ezildi.

**Ölçüm 2 — CAT_7 bağlam-sızıntı (semantik kontaminasyon):**
- Üyeler: `süt, tenis, çayı, su, yüzme, hava, ...+27` (33 toplam)
- Ek imzası: 56+ ayrık son-ek, normalize homogeneity ≈ **0.97** (ekstrem)
- Domain karışımı: **içecek** + **spor** + **hava-durumu** aynı kovada
- Mekanizma: koşullu cümlelerin sonuç kısmı ("yorulursa su iç", "hava ısınırsa")
  bu üçünü aynı bağlam-vektöründe buluşturdu
- `CAT_21`den farklı hata: orada morfolojik çakışma, burada **semantik sızıntı**

**Ölçüm 3 — SPEAK koşul tetikleme (kullanım testi):**

| Sorgu | Çıktı | Koşul eki? |
|---|---|---|
| kar yağarsa ne olur | `kar soğuk ıslatıyor` | ❌ |
| kedi açsa ne yapar | `kedi basketbol okur` | ❌ |
| çocuk okula gitmezse ne olur | `okul soğuk ol` | ❌ |
| hava soğuk olursa ne yaparsın | `hepimiz havacılık yeriz` | ❌ |

**Tetikleme oranı: 0/4 = %0.** Demo [09] `kedi uyanırsa okur` istisnaydı (rastlantısal).

#### 8.8.3 Kapanış Teşhisi

> **"Temsil var, kullanım yok."**

- **Kazanım**: `-sa/-se` morfolojisi **representation** seviyesinde keşfedildi
  (CAT_14 mini'de 0.94, CAT_18/21 full'da mevcut). Cluster mekanizmasının
  morfolojik ekleri yakalayabildiği kanıtlandı.
- **Başarısızlık**: Kategori tabanlı koşul öğrenme **QA'ya aktarılamadı**.
  QA −11 puan regresyon, SPEAK tetikleme %0. Koşul bir **kategori üyeliği değil,
  ilişki türü**.

#### 8.8.4 Öğrenilen Ders (Aşama 3'e Girdi)

Distributional clustering **yapısal rolleri** (fiil, nesne, mekan) iyi ayırıyor
ama **semantik fonksiyonları** (koşul, neden, zaman) ayıramıyor. Bu fonksiyonlar
**graf düzleminde** temsil edilmeli — kategori düzleminde değil.

Aşama 3 (Sitoiskelet) gerekçesi sayısal olarak sertleşti: koşul bir kenar türü
(`Q_conditional → A_conditional`) olarak modellenmeli, üyelik değil.

#### 8.8.5 Rollback (14 Nisan, 12:33)

- `train_data_15k.txt`: 15600 → **13600 satır** (pre-S2)
- `ribo31.bin` MD5 = `ribo31_v72_muhur.bin` MD5 (baseline mührü aktif)
- Arşivlendi: `train_data_15k_s2_archive.txt`, `ribo31_s2_archive.bin`, `s2_full.log`
- Korundu: `conditional_train.txt`, `conditional_test.txt`, `gen_conditional.py`
  (Aşama 3'te Sitoiskelet **ilişki etiketi kaynağı** olarak yeniden kullanılacak)

#### 8.8.6 Sonraki Adım

Aşama 2.5 — **Güven Altyapısı (Confidence Substrate)**. Dirty Cluster Detector'ın
ilk gerçek vaka çalışması CAT_21 verisi (homogeneity 0.93, transition entropy %28).

> *Aşama 2 başarısız değil — **bilişsel bir sınır kanıtı**. Başarısızlık
> haritalandı, Aşama 3 gerekçesi sayılarla belgelendi, baseline korundu.*

---

## 9. Aşama 2.5 — Güven Altyapısı (14 Nisan, 12:45 →)

Tam tasarım: `ASAMA_2_5_TASARIM.md`. Burada özet + anahtar bulgular.

### 9.1 Faz A — Dirty Cluster Detector (Tamamlandı, 14 Nisan 13:15)

**Modül:** `ribozom_cluster_qa.c/h`, main'e `--cluster-qa` flag.
**Metrikler:** `trust = geomean(cohesion, wpos_homogeneity, suffix_homogeneity)`.
**Binary:** `ribozom_v31_v6.exe` (docker-desktop musl-gcc).

#### 9.1.1 v7.2 Baseline Raporu (`cluster_qa_baseline.log`)

| Sınıf | Sayı | % | Anlam |
|---|---|---|---|
| CLEAN (trust ≥ 0.70) | **0** | **%0** | Hiçbir kategori "tam güvenilir" değil |
| MIXED (0.40–0.70) | 9 | %41 | Sınırlı kullanılabilir |
| DIRTY (< 0.40) | 4 | %18 | Aşama 3 kenar kaynağı **değil** |
| SMALL (n < 5) | 9 | %41 | Veri yetersiz |

**NCAT=22 (baseline), 25 (Aşama 2) — koşul verisi +2 DIRTY (CAT_21 tipi) ekledi.**

#### 9.1.2 Baseline Kritik Bulgular

- **CAT_0 DIRTY (trust=0.36)** — Özne/BOS sınıfı, n=243. coh=0.85 (iyi), ama
  wpos_hom=**0.34** (heterojen), sfx_hom=0.16 (99 ayrık son-ek). Her domainden özne
  aynı kovada → BOS pozisyonu çok geniş.
- **CAT_1 MIXED (trust=0.59)** — Ana fiil sınıfı, n=248. coh=0.96, wpos B4=%81
  (doğru, son pozisyon). sfx_hom 0.31 çünkü -or/-yor/-ır/-er/-dı/-di Türkçe
  morfolojik varyantları ayrı son-ek sayılıyor (metrik rafinasyonu gerekli).
- **CAT_14 DIRTY (trust=0.16)** — "blok, antrenman, skor, tenis, yüzme, koşu,
  basketbol". Aşama 2'deki CAT_7 semptomunun baseline'daki yansıması. Spor +
  ölçüm karışımı.

Dominant bucket dağılımı: `B0=1 B1=3 B2=9 B3=8 B4=1`. B4 (fiil) sadece CAT_1'e
argmax — fiiller tek kümede toplanmış (beklenen).

#### 9.1.3 Ana Bulgu: "Baseline Zaten Kirli"

> Baseline'da **%0 CLEAN** kategori var. Bu bir başarısızlık değil, bir
> **görünürlük kazanımı**. Aşama 3 Sitoiskelet'in her kategoriye `trust`
> ağırlığıyla yaklaşması gerektiğini sayısal olarak kanıtlıyor. CAT_0'dan
> (0.36) gelen kenar ile CAT_1'den (0.59) gelen kenar aynı ağırlıkta
> olmamalı.

Bu bulgu Aşama 3 tasarımını değiştiriyor: `causal_graph_t.f_weight[]` tek başına
yeterli değil, `trust[source_cluster] * trust[target_cluster]` çarpanı eklenmeli.

#### 9.1.4 Bilinen Metrik Sınırları (Yol 2 — Planlandı)

`suffix_homogeneity` Türkçe morfoloji-bilinçli değil:
- `-dı/-di/-du/-dü/-tı/-ti/-tu/-tü` hepsi "geçmiş zaman 3.tekil" ama 8 ayrık
  son-ek olarak sayılıyor → yapay heterojenite
- Rafinasyon: ünlü uyumu sınıfı + sert/yumuşak ünsüz normalizasyonu
- **Faz B blocker değil** — mevcut ham metriklerle Faz B yürür, rafinasyon
  sonradan düşer

#### 9.1.5 Metodoloji Kararları

- **Yol 1 REDDEDİLDİ** — eşikleri "daha güzel görünsün" diye 0.70→0.55'e çekmek
  sinyal kaybı, "kendimizi kandırmayalım" ilkesi ihlali
- **Yol 3 KABUL** — detector olduğu gibi kabul, baseline kirli bulgusu Aşama 3
  girdisi
- **Yol 2 PLANLANDI** — Faz B sonrası morfoloji-bilinçli suffix rafinasyonu

### 9.2 Faz B — Unified Confidence (Tamamlandı, 14 Nisan 13:40)

**Modül:** `ribozom_confidence.c/h`, main'e `--confidence-demo` flag.
**Mod:** PASİF — hesaplar, loglar, QA pipeline'ına dokunmaz.

#### 9.2.1 Mimari

```c
typedef struct {
    float pattern_conf;   /* cluster cohesion (cats[src].coh) */
    float position_conf;  /* predicted_wid wpos dominant share */
    float category_conf;  /* 1.0 pass-through (Faz C/D'de aktif) */
    float domain_conf;    /* 1.0 pass-through (Faz C Absurd Detector) */
    float trust_source;   /* Faz A cache: cluster_trust_of(src) */
    float aggregate;      /* geomean5, eps=0.01 */
} confidence_t;
```

**Yenilik:** `trust_source` — Faz A'daki cluster quality doğrudan
confidence'a akıyor. Düşük-trust kategoriden gelen tahmin, pattern_conf
yüksek olsa bile geomean ile bastırılır.

#### 9.2.2 Demo Sonucu (10 sorgu, `confidence_demo.log`)

| Grup | Sonuç | Yorum |
|---|---|---|
| GOOD 5/5 | ✅ AGG 0.62–0.77 | İyi sorgular eşiği geçiyor |
| BAD 2/5 | ⚠ Sadece OOV yakalandı | "taş uyur", "5 yatar" — src=-1 |
| BAD 3/5 | ❌ AGG 0.71–0.76 | "ağaç koşar", "kedi uçak okur" — yakalanamadı |

#### 9.2.3 Sınır ve Faz C Gerekçesi

Faz B pasif pass-through olduğu için `category_conf=1.0` ve `domain_conf=1.0`.
Pattern+Position+Trust tek başına **type-mismatch** (cansız özne + canlı fiil)
ve **cross-domain** (kedi + uçak) kombinasyonlarını ayıramıyor.

Ek bulgu: **Tüm içerik kelimeleri CAT_0'a düşüyor** (243 üyeli geniş özne
sınıfı, trust 0.36). "ağaç" da "kedi" de "taş" da aynı kovada. Bu Faz A'daki
*"CAT_0 DIRTY"* bulgusunun pratik yansıması — source cluster tek başına
semantik sinyal taşımıyor.

#### 9.2.4 Faz C Tasarım Kısıtı (F.2 anti-kalıp koruması)

**REDDEDİLEN yaklaşım:** canlı/cansız ontoloji hardcode. Bu özel-case
yazma, AGI_YOL_HARITASI §F.2 ihlali.

**KABUL EDİLEN yaklaşım:** 3 kural, hepsi mevcut veri yapılarından:

1. **Unknown content ratio** — içerik kelimelerinin OOV oranı > %50
2. **Cluster-trans boşluğu** — `trans[src][dst]/trans_row_total[src]<0.005`
   (Mikrotubul "bu geçişi hiç görmedim" sinyali, istatistik)
3. **Cluster-tip ikili** — query'nin iki içerik kelimesinin dominant wpos
   bucket'ı aynı (iki B0 özne veya iki B4 fiil → "kedi uçak" yakalanır)

Üçü de ontoloji-free, mevcut `trans[][]`, `wpos[][]`, `wf()`, `word_cat()`
API'leri ile çalışır.

#### 9.2.5 Faz B Kabul Kriteri

- [x] 5-boyut confidence hesaplanıyor ve loglanıyor
- [x] Bilinen iyi sorgular aggregate ≥ 0.5 (5/5 geçti)
- [x] Bilinen kötü sorgular AGG ≤ 0.3 (sadece OOV — kısmen; Faz C tamamlayacak)
- [x] QA baseline metriklerinde sıfır değişim (pasif mod doğrulandı)
- [x] trust_source Faz A'dan kanalize ediliyor

### 9.3 Faz C — Absurd Detector (Tamamlandı, 14 Nisan ~14:10)

Spec: `ASAMA_2_5_TASARIM.md §4`. Ontoloji-free 3 kuralla başlandı, K3
kapanışta kaldırıldı.

**Kurallar (Final):**
- **K1 — Unknown Content Ratio** (ağırlık 0.55): içerik token'larında OOV
  oranı > %50 ise tetiklenir. Rakam-yalnız ve bilinmeyen kelimeli
  sorgularda isabetli.
- **K2 — Cluster-Trans Boşluğu** (ağırlık 0.45): Mikrotübül `trans[][]`
  matrisinde src→dst geçiş oranı < %0.5 ise tetiklenir. `trans_row_total
  < 20` ise istatistik güvenilmez, kural atlanır (tanılamada `trans_skipped=1`).
- **K3 — Cluster-Tip İkili** → **KALDIRILDI**.

**K3 Kaldırma Gerekçesi (karar ~14:05):**
`abs_wpos_argmax()` cluster-wide dominant bucket döndürür, cümle-bağlamı
değil. Demo sonucunda "kedi balık yer" gibi GOOD cümlelerde balık da
cluster-wide B0-dominant olduğu için false positive üretti (dom 1.00 →
0.75). False-positive zararı, real-positive faydayı aştı. Mevcut veri
yapılarıyla (wpos, trans, word_cat) ontolojik "rol çakışması"
güvenilir çıkarılamıyor — sentence-scoped cümle-pozisyonu bilgisi yok.
Bu sinyal Aşama 4'te (ontoloji) veya Faz A+ aşamasında
(KL-divergence varyantı) geri gelebilir.

**Birleşik Skor (K3 sonrası):**
```
absurd_score = 0.55·[K1] + 0.45·[K2]        # [·] ∈ {0,1}
domain_conf  = 1 - absurd_score             # clamp [0,1]
```

**Demo Sonuçları (10 sorgu, K3'süz):**

| Grup | query | pat | pos | dom | trust | AGG | verdict |
|------|-------|-----|-----|-----|-------|-----|---------|
| GOOD | kedi balık yer | 0.85 | 0.77 | **1.00** | 0.36 | 0.75 | OK |
| GOOD | kuş uçar | 0.85 | 0.83 | **1.00** | 0.36 | 0.76 | OK |
| GOOD | doktor hasta bakar | 0.85 | 0.30 | 0.45 | 0.36 | 0.53 | OK |
| GOOD | öğretmen ders anlatır | 0.85 | 0.79 | **1.00** | 0.36 | 0.75 | OK |
| GOOD | çocuk oyun oynar | 0.85 | 0.88 | **1.00** | 0.36 | 0.77 | OK |
| BAD | taş uyur | 0.30 | 0.30 | 0.45 | 0.00 | 0.20 | OK |
| BAD | ağaç koşar | 0.85 | 0.81 | 1.00 | 0.36 | 0.76 | HIGH |
| BAD | kedi uçak okur | 0.85 | 0.76 | 1.00 | 0.36 | 0.75 | HIGH |
| BAD | 5 yatar | 0.30 | 0.30 | 0.45 | 0.00 | 0.20 | OK |
| BAD | balık yüzer ağaç | 0.85 | 0.58 | 1.00 | 0.36 | 0.71 | HIGH |

**Özet: GOOD 5/5, BAD 2/5.**

**Kritik Bulgu — Ontoloji Sınırı:**
K3 öncesi BAD 2/5 idi; K3 dahil edildiğinde "kedi uçak okur" yakalanıyordu
ama "kedi balık yer" (GOOD) de yanlış yakalanıyordu. K3 kaldırılınca
GOOD 5/5 kurtarıldı, BAD skoru değişmedi. Gerçek fark: "kedi balık yer"
de dahil 4 GOOD cümlenin `dom=1.00`'e geri dönmesi.

Yakalanamayan 3 BAD sorgunun ortak özelliği:
- `ağaç koşar`: her iki kelime bilinen, CAT_0→CAT_1 yaygın geçiş
  (canlı→fiil şablonu), K2 tetiklemiyor.
- `kedi uçak okur`: tüm kelimeler bilinen, 3 içerik ama hiçbir kural
  *pair-wise* semantik uyumsuzluğu görmüyor.
- `balık yüzer ağaç`: yapı bozuk ama kelime düzeyinde anomali yok.

Bu üç örnek, "kelime anlamı / varlık tipi" bilgisi gerektiriyor. Mevcut
veri yapıları (frekans, wpos, trans, kategori kohezyonu) bu sinyali
taşımıyor. **Bu Aşama 4'ün (ontoloji) işi, Aşama 2.5'in değil.**

**F.2 Anti-Kalıbı Koruması:** Kural setine "canlı/cansız" veya özel
nesne listeleri eklemek teklif edilmedi. "Özel case yazma" kuralı
korundu; sınır dürüstçe belgelendi.

**Faz C Kapanış Statüsü:**
- Motor çalışıyor, pasif mod korundu (pipeline etkilenmedi).
- K1+K2 "absurd telemetri" olarak anlamlı sinyal veriyor (taş/5 yatar
  skoru 0.20'de net ayrışıyor).
- Ontolojik absurd yakalanması **bilerek** ertelendi — kalibrasyon
  Aşama 4 (CSR + lexical semantics) zemininde yapılacak.

**Artifact'lar:**
- `ribozom_absurd.h`, `ribozom_absurd.c` (K3 placeholder'lı, her zaman 0
  döndürür; telemetri alanı `role_collision_flag` struct'ta korundu,
  geriye-uyum için).
- `confidence.domain_conf` kanalı: `1 - absurd_score`.
- Demo binary: `ribozom_v31_v6.exe --confidence-demo`.

**Karar Kayıtları:**
- 14:05 — K3 kaldırma (GOOD false positive, opsiyon 1 kabul).
- 14:10 — Faz C kapanış, Faz D'ye geçiş onayı.

### 9.4 Faz D — Calibration Telemetry (Kapandı ⚠️, 14 Nisan 14:30)

Amaç: `confidence_t.aggregate` ↔ gerçek doğruluk kalibrasyonunu ölç
(10-bin reliability diagram + tek sayı ECE). Mevcut test setiyle (500
test + 100 OOV) pasif modda.

**Artifact'lar:**
- `ribozom_calibration.h`, `ribozom_calibration.c`
- `--ece` flag'i main.c'ye eklendi (pasif mod, pipeline'ı bozmuyor)
- `run_calibration_report(lines, n, label)` fonksiyonu

**Metodoloji:**
- Her test satırı için `compute_confidence(text, wf(last_word), -1).aggregate`
- Accuracy proxy: son kelimenin TÜM karakterlerinin `predict_meta_v31`
  ile doğru tahmini (tam-kelime binary)
- 10-bin ECE = Σ (n_b/N) · |avg_conf_b − accuracy_b|

**İlk Sonuçlar (son-karakter proxy) → İkinci Koşu (tam-kelime proxy):**

| Set | accuracy | avg_conf | ECE | dağılım |
|-----|----------|----------|-----|---------|
| test_data_15k (500) | 0.996 | 0.624 | **0.3716** | 500/500 → bin [5–7] |
| oov_test_15k (100) | 1.000 | 0.233 | **0.7671** | 100/100 → bin [2] |

Proxy değişimi accuracy tavanını kırmadı (teacher forcing + Türkçe
ek-morfem öngörülebilirliği). Ama asıl bulgu proxy değil.

**Kritik Bulgu — Flat Confidence Dağılımı:**
500 test örneğinin **tamamı** tek bin'e (0.6–0.7) düşüyor. 100 OOV'nin
**tamamı** bin [2]'ye (0.2–0.3) yapışıyor. Aggregate, örnekler arası
**ayrımcılık gücü üretmiyor**.

**Kök Neden — geomean5 bileşen analizi:**

| Bileşen | Test dağılımı | OOV dağılımı | Çeşitlilik |
|---------|---------------|--------------|------------|
| pattern_conf | ~0.85 (CAT_0 kilidi) | 0.30 (default) | yok |
| position_conf | 0.3–0.9 | 0.3 (default) | tek gerçek değişken |
| category_conf | 1.0 sabit | 1.0 sabit | Faz D'de aktif olacaktı — kalibrasyonun ön koşulu zaten sağlanamadı |
| domain_conf | 1.0 (K1/K2 tetiklemez) | 0.45 (K1 tetik) | ikili |
| trust_source | 0.36 (CAT_0 kilidi) | 0.00 (src=-1) | ikili |

**Yapısal sonuç:** confidence_t.aggregate bir **sıralama sinyali değil**,
source-known/source-unknown ikili göstergesi. Flat dağılım kozmetik
bir formül hatası değil — Faz A'da tespit edilen **CAT_0 kilidinin**
doğrudan yansıması. Tüm içerik kelimeleri tek kategoriye yığılıyor
→ pattern_conf ve trust_source sabit → geomean tek bin'e kilitleniyor.

**ECE 0.37 ve 0.77 yorumu:** Sayısal olarak "ciddi underconfidence"
gibi görünse de gerçek anlam farklı — kalibrasyonun önkoşulu
(confidence'ın ayrışması) sağlanmadığı için ECE ölçümü anlamlı bir
kalibrasyon hatası değil, **aggregate'in bilgi taşımadığının** sayısal
kanıtı.

**Kapatma Gerekçesi (14:30):**
Formülle oynamak (örn. trust'ı çarpan yapma, position_conf ağırlığını
büyütme) kozmetik düzeltme olur. Kök neden CAT_0 kilidi ve flat trust
dağılımı — bunları gidermek **Aşama 3 Sitoiskelet'in işi** (CSR tabanlı
semantik komşuluk, trust-weighted kategori ayrışması).

**Statü:** ⚠️ Motor çalışıyor, telemetri üretiyor, ama kalibrasyon
hedefine (ECE < 0.1) mevcut veri yapılarıyla ulaşılamıyor.

### 9.5 Faz E — Test Seti + Kapanış Sinyali (Ertelendi ❌, 14 Nisan 14:30)

**Planlanan:** 20 absurd cümle + 20 known-good test seti, ECE<0.1 ile
"kapanış sinyali" + Aşama 2.5 resmi kilidi.

**Erteleme Gerekçesi:** Faz D bulgusu ışığında ECE<0.1 hedefi mevcut
veri yapılarıyla karşılanamaz. Test setini genişletmek aggregate'in
flat dağılımını değiştirmez. Kapanış sinyali Aşama 3 Sitoiskelet
sonrasına taşındı.

**Yeniden aktive olma koşulu:** Aşama 3'te CAT_0 kilidi kırıldıktan
(CSR kenarları + trust-weighted komşuluk) ve kategori ayrışması
sayısallaştıktan sonra Faz E doğrudan koşulabilir — motor zaten hazır.

### 9.6 Aşama 2.5 RESMİ KAPANIŞ (14 Nisan 14:30)

**Özet Tablo:**

| Faz | Statü | Çıktı |
|-----|-------|-------|
| A — Dirty Cluster Detector | ✓ | Motor çalışıyor, baseline %0 CLEAN bulgusu |
| B — Unified Confidence | ✓ | geomean5 motoru, trust_source kanalı, pasif mod |
| C — Absurd Detector | ✓ | K1/K2 aktif, K3 kaldırıldı, ontoloji sınırı belgeli |
| D — Calibration Telemetry | ⚠️ | ECE hesaplanıyor, aggregate flat — ayrışma yok |
| E — Test seti + kapanış | ❌ | Ertelendi, Aşama 3 sonrasına |

**Aşama 2.5 Kazanımı:**
- Confidence altyapısı **kuruldu**: 5-kanallı geomean5 motoru, trust
  cache, absurd detektör, ECE telemetri — tümü pasif modda pipeline'ı
  bozmadan çalışıyor.
- Sınırları **keşfedildi**: ontoloji gerektiren semantik absurd
  yakalanamıyor (F.2 korundu, Aşama 4'e), confidence aggregate
  CAT_0 kilidi nedeniyle ayrışmıyor.
- **Aşama 3 Sitoiskelet'in gerekliliği sayısal olarak kanıtlandı**:
  Faz A'dan %0 CLEAN, Faz D'den flat ECE — ikisi de aynı kök nedeni
  gösteriyor (trust-weighted kategori ayrışmasının yokluğu).

**Mühür (14 Nisan 14:30):**
Motor hazır; yakıt Sitoiskelet'ten gelecek. Aşama 2.5 kapandı,
Aşama 3'e geçiliyor.

**Artifact Envanteri:**
- `ribozom_cluster_qa.{h,c}` (Faz A)
- `ribozom_confidence.{h,c}` (Faz B)
- `ribozom_absurd.{h,c}` (Faz C)
- `ribozom_calibration.{h,c}` (Faz D)
- `ASAMA_2_5_TASARIM.md` — spec + karar kayıtları
- `ribozom_v31_v6.exe` — `--cluster-qa`, `--confidence-demo`, `--ece` flag'leri
- `PROJE_DURUMU_v15K.md §9` — tam rapor

**Karar Kayıtları (Faz D kapanış):**
- 14:25 — Faz D ilk koşu sonuçları: ECE 0.37/0.77, flat dağılım tespiti.
- 14:28 — Proxy değişimi (son-char → tam-kelime) accuracy tavanını kırmadı.
- 14:30 — Faz D flat-distribution kapanışı kabul; Faz E ertelendi;
  Aşama 2.5 resmi kapanış imzalandı ("Motor hazır; yakıt
  Sitoiskelet'ten gelecek.").

---

## §10 — AŞAMA 3: SİTOİSKELET / KAUSAL GRAF

**Hedef:** Kelime-seviyesinde koşul→sonuç ilişkilerini kategori
düzleminden izole bir grafta tutmak. Aşama 2'nin dersi: koşul verisi
kategori düzlemine karıştığında QA -11 puan düşer. Bu nedenle kausal
graf **ayrı düzlem** olarak inşa edildi — `rtrain / sent_store /
trans` pipeline'ına kesinlikle dokunmuyor.

**Mimari özeti:**
- Dual CSR (forward cause-indexed + backward effect-indexed)
- Arena allocation: node başına `EDGES_PER_NODE=32` statik slot,
  realloc yok
- LWU (Least Weight Used) evict — cap aşılırsa en zayıf kenar atılır
- Bounded BFS, bitmap visited, `CAUSAL_DEPTH_LIMIT=3`, `HOP_DECAY=0.7`
- Konfidans formülü: `conf = Π(w_i) × decay^(depth-1)`

### 10.1 Faz A — Veri Yapısı
**Çıktı:** `ribozom_causal.{h,c}`, 20/20 sanity PASS
(init/add/has/degree/duplicate/self-loop/save/load roundtrip).
Mevcut pipeline'a etki yok (doğrulandı: `--causal-stats` boş graf,
500-test/100-OOV baseline bozulmadı).

### 10.2 Faz B — causal_observe
**Trigger gramer:** "-sa/-se" koşul eki, UTF-8 byte-wise suffix
matching (12 varyant: ırsa, ürse, arsa, erse, irsa, irse, ursa,
urse, rsa, rse, sa, se; min stem 3 byte).

**Kritik karar:** Trigger **operatör**, kenar **varlık→varlık**.
Yani "ateş yaksa yemek pişer" → edge = (ateş, yemek), trigger
kelimesi ("yaksa") kenar ucu değil; iki içerik kelimesini bağlayan
işaret.

**Stopword v1 (ilk dalga):** 35 kelime (zaman zarfları, sorular,
bağlaçlar, zamirler) — ilk dump'ta "sonra" 8/20 edge'i (%40 gürültü)
dominasyonunu kırdı.

**Eğitim sonucu:** `conditional_train.txt` → **59 unique edge**,
19 aktif node, 0 eviction, top-20'de 0/20 saçma kenar.

### 10.3 Faz C — BFS + CLI
**API:** `causal_reachable(src, dst, direction, out)` bounded BFS,
parent-chain path reconstruction; `causal_query_cli` formatlı
yazdırma.

**CLI flag'leri:** `--causal-query`, `--causal-query-back`,
`--causal-dump N`.

**Validasyon (9/9 PASS):**
- 5 direkt edge sorgusu (FWD+BWD), hepsi depth=1 FOUND
- 3 chain-2 sorgusu (köpek→çayı, kedi→çayı, kar→bir), hepsi depth=2
  FOUND, conf formülü doğrulandı
- 1 negatif kontrol (bebek→nehir) → doğru şekilde NOT REACHABLE

### 10.4 Faz D — Graf Kalitesi
**Stopword v2 (ikinci dalga):** "bir", "en", "de", "da" eklendi.
"bir" 1 edge'i sildi (59→58), "kar→hava→bir" zinciri doğru şekilde
çözüldü.

**Evidence-based weight:** Flat 0.50 → dinamik formül
`w = min(1.0, 0.3 + 0.05 × evidence)`. Sonuç:
- ateş→yemek (ev=13) → **0.95** (güçlü)
- yemek→çayı (ev=9) → **0.75**
- bebek→süt (ev=8) → **0.70**
- kedi→yemek (ev=2) → **0.40** (zayıf)
- Avg 0.500 → 0.538; range 0.35–0.95

Caller override korundu (sanity test bozulmadı, 20/20 hâlâ PASS).

### 10.5 Faz E — Kapanış Test Seti

**30 sorgu, eşik ≥27/30. Sonuç: 30/30 (100%).**

| Grup | Sorgu | Geçen |
|------|-------|-------|
| A — FWD direkt | 10 | 10/10 |
| B — BWD direkt | 10 | 10/10 |
| C — Chain-2 | 5 | 5/5 |
| D — Negatif (sink) | 5 | 5/5 |

**Kalite indikatörleri:**
- Confidence ayrışması: 0.550 → 0.950 aralığı (dinamik)
- Chain-2 decay doğrulaması: köpek→yemek→çayı = 0.65×0.75×0.70 =
  0.341 ✓
- FWD/BWD simetri: A01 (ateş→yemek) ve B01 (yemek→ateş) aynı
  conf=0.950 — dual CSR senkronu sağlam
- Negatif kontrol: 5/5 sink-node sorgusu NOT REACHABLE, false-positive
  yok

### 10.6 Aşama 3 RESMİ KAPANIŞ

**Artifact Envanteri:**
- `ribozom_causal.{h,c}` — dual CSR + BFS + serialize
- `conditional_train.txt` — eğitim korpusu
- `ribo31_causal.bin` — 58 edge, 18 node mühürlü graf dosyası
- CLI flag'leri: `--causal-sanity`, `--causal-train`,
  `--causal-query`, `--causal-query-back`, `--causal-dump`
- `_causal_test.sh`, `_causal_2hop.sh`, `_causal_faz_e.sh` —
  regression test skriptleri

**Ayrı düzlem koruması doğrulandı:** Aşama 3 boyunca `rtrain`,
`sent_store`, `trans`, `wpos` veri yapılarına tek bir yazma işlemi
yapılmadı. 500-test / 100-OOV baseline skorları değişmedi.

**Kazanım:**
- Aşama 2.5'te "motor hazır, yakıt yok" durumu vardı; Aşama 3'te
  **yakıt geldi**: kelime-seviyesi kausal bağlar graf olarak mühürlü.
- BFS + evidence-based confidence ile çok-hop çıkarım artık mümkün:
  "ateş→yemek→çayı" conf=0.341 ile traversable, false-positive
  üretmeden.
- Dün: "kartal eti yer" (trans tablosuyla bir-adım). Bugün:
  "köpek→yemek→çayı" (iki-adım kausal zincir, güven skoru, path
  reconstruction).

**Bilinen sınırlar (Aşama 4+ konusu):**
- Morfoloji: "çayı" (accusative) → "çay" normalizasyonu yok.
- Olumsuzluk / polarite: "ateş soğuksa yemek pişmez" gibi
  polarite-ters zincirler henüz yok (**Aşama 4**).
- Graf yoğunluğu seyrek (58 edge/18 node) — 3+ hop zincirleri
  kesişim azlığı nedeniyle nadir.
- "çünkü" koşul-olmayan kausal ibare parser'ı henüz yok.

**Mühür:**
Aşama 3 Sitoiskelet 30/30 ile kapandı. Kausal düzlem bağımsız,
doğrulanmış, evidence-weighted. Aşama 4 (Olumsuzluk ve Polarite)
için altyapı hazır — ilk kez "yokluk" eklenecek; taze kafa lazım,
durulacak.

Aşama 3 resmi kapanış saati: 15:45.

---

## 11. Aşama 4 — Olumsuzluk ve Polarite (14 Nisan, 16:00 → 17:30)

### 11.1 Tasarım Kararları (Mühürlü)
Üç seçenek arasından **signed edge** (seçenek 2) seçildi:
- Polarite **edge'e** aittir, node'a değil. "ateş enabling" değil,
  "ateş→yemek +1", "ateş→yemek -1" ayrı kayıtlar.
- Ayrı negatif graf değil — tek graf, slot anahtarı `(src, tgt, polarity)`
  üçlüsü. `causal_add_edge(cause, effect, weight, polarity)`.
- Kanonikleştirme: `polarity = (cause_neg == effect_neg) ? +1 : -1`.
  Contrapositive ("¬A → ¬B") ve vanilla ("A → B") aynı kanonik polariteye
  düşer. Bu Türkçe'deki "-mazsan ... -mez" kalıbını ontoloji olmadan
  çözer.
- "değil" / "yok" (free-standing particle) **Faz B.5'e ertelendi** —
  bağlam-bağımlı, parser'ı ikiye katlıyor.

### 11.2 Faz A — Struct Genişletme
- `causal_graph_t` için `signed char f_polarity[MAX_CAUSAL_EDGES]`
  ve `b_polarity[...]` eklendi; dual CSR her iki yönde senkron polarite
  taşıyor.
- `causal_has_edge` artık `(cause, effect, polarity)` üçlü anahtar
  alıyor; `cau_find_fwd_slot` aynı şekilde.
- Save format v2 — polarite dizileri dosyaya eklendi; v1 legacy load
  default `+1` ile uyumlu geriye uyumluluğu koruyor.
- Sanity 20 → **32/32 PASS** (12 polarite-özel test).

### 11.3 Faz B — Negation Parser
- Türkçe olumsuzluk eki tarayıcı: `cau_has_neg_marker()` —
  `-ma/-me + z` (aorist-neg) substring detektörü, token başında aramaz
  (kök uyumuna saygı).
- Privative suffix: `-sız/-siz/-suz/-süz` (nominal yokluk)
  `cau_try_privative` ile ayrıştırılıyor.
- `CAUSAL_SUFFIXES` 2nd person formlarıyla genişletildi
  (`-mazsan/-mezsen` 295 cümlede baskın).
- Eğitim sonucu: 60 edge, **25 cause-neg** (10%), 225 polarity=+1,
  25 polarity=-1.
- Korpus sınırı: effect-neg aktif cümlelerde parse pozisyonu OOV'a
  denk geldi (plural/soru formları) — 0 doğal effect-neg edge.
  Mekanizma çalışıyor, data ince.

### 11.4 Faz C — BFS Polarite Çarpımı
- Scratch buffer: `cau_edge_pol[MAX_CAUSAL_NODES]`.
- Path reconstruction sırasında `pol_prod *= cau_edge_pol[cur]`,
  `out->polarity = sign(Π p_i)`.
- CLI çıktısı: `FOUND depth=1 polarity=- NEG (preventing) confidence=1.000`.
- Kritik doğrulama: `erken → geç` → `polarity=-`, `geç → erken` (BWD)
  → `polarity=-` (dual CSR senkron kanıtı).

### 11.5 Faz D — K3 polarity_collision (reborn)
- Eski K3 (wpos-argmax role_collision) 2.5 Faz C'de ontoloji gerektirdi,
  kaldırılmıştı. Aşama 4'te **veri-yerel** biçimde yeniden doğdu:
  aynı `(src, tgt)` için hem `+` hem `-` edge varsa → eğitim verisinde
  çelişki → absurd sinyali.
- Ağırlıklar: K1=0.45, K2=0.35, **K3=0.20** (toplam 1.00).
- Sanity 3/3 PASS: yalnız(+) → K3=0; (+,-) ikisi → K3=1; farklı çift →
  K3=0.

### 11.6 Faz E — Kapanış Testi
30 veri-kaynaklı sorgu + 3 K3 sanity = **33/33 PASS**:
- Grup A (10 FWD pozitif): 10/10
- Grup B (5 BWD pozitif): 5/5
- Grup C (5 chain-2 pozitif): 5/5
- Grup D (5 NOT REACHABLE): 5/5
- Grup E (5 polarite: neg FWD/BWD + 3 kontrol): 5/5
- K3 polarity_collision sanity: 3/3

### 11.7 Aşama 4 RESMİ KAPANIŞ

**Artifact Envanteri:**
- `ribozom_causal.{h,c}` — polarity dual CSR, v2 save format
- `ribozom_absurd.{h,c}` — K3 polarity_collision (reborn)
- CLI: `--absurd-k3-test` (Faz D sanity)
- `_asama4_faz_e.sh` — regresyon scripti

**Organel haritası güncel:**
- Polarite Alanı (Membrane charge) 📋 → **✅**

**Kazanım:**
İlk kez "yokluk" (¬A) sisteme girdi. Contrapositive Türkçe koşul
kipi (`-mazsan ... -mez`) canonical polariteye eşlenebiliyor.
BFS'te polarite zincir boyunca çarpıldığı için, "A prevents B,
B prevents C, so A enables C" (- × - = +) otomatik çıkıyor.

**Aşama 4 resmi kapanış saati: 17:30.**

---

## 12. Entegrasyon Borcu (Mühürsüz — Sonraki İş)

**14 Nisan itibariyle:** Aşama 1, 2.5, 3, 4 hepsi kendi test setlerinde
temiz. **Ama birbirlerinden habersiz çalışıyorlar.**

- SPEAK pipeline (v7.2) kausal graf'ı konsülte etmiyor.
- Confidence Substrate, kausal sorgu sonucunu etkilemiyor.
- "Kar yağarsa ne olur?" SPEAK'te hâlâ trans/wpos'tan üretiliyor;
  kausal grafta potansiyel olarak `kar → evde` edge'i var ama
  dokunulmuyor.
- Polarite SPEAK çıktısında yansımıyor — "yağmazsa" sorusu pozitif
  cevap üretebiliyor.

**Risk:** Her aşama hızlı geçtiği için modüller birbirine bağlanmamış.
15 aşamanın sonunda bunlar entegre edilmezse **15 bağımsız modül**
kalır, tek bir AGI gibi davranan sistem değil.

**Karar (beklemede):** Aşama 5'e geçmeden önce **Aşama 3+4'ü SPEAK'e
bağlama** işi yapılmalı. Yol haritasında bu bir "ara aşama" değil,
bir **bütünleşme borcu** olarak mühürlenmeli.

**Karar Kayıtları:**
- 14:45 — Faz A sanity 20/20; boş graf `--causal-stats` temiz.
- 14:55 — Faz B ilk dump: "sonra" %40 dominasyon → stopword v1.
- 15:05 — Faz B temiz dump (0/20 saçma), 59 edge.
- 15:15 — Faz C 5/5 direkt sorgu FOUND, ama tümü depth=1 (zincir
  tetiklenmedi).
- 15:20 — 2-hop zorla test: 3/3 chain + 1/1 negatif, 9/9 total.
- 15:30 — Faz D evidence-based weight + stopword v2; conf
  ayrışması 0.55→0.95 dinamik.
- 15:45 — Faz E 30/30 kapanış; Aşama 3 mühürlendi.
  Sitoiskelet'ten gelecek").

**BORÇ ÖDENDİ — 14 Nisan 19:xx:** Aşama 4.5 (SPEAK × Kausal Köprü)
tamamlandı. Modüller artık konuşuyor. §13'e bakınız.

---

## 13. Aşama 4.5 — SPEAK × Kausal Entegrasyonu (14 Nisan, 18:00 → 19:xx)

**Amaç:** Aşama 3 (kausal graf) + Aşama 4 (polarite) mevcut SPEAK
pipeline'ına **additive** bağlanır. Baseline (Test 58.2%, v5.0 QA
65.9%) **değişmez**; sadece üç yeni soru formu kausal cevap üretir.

### 13.1 Kapsam

**Kapsam-içi (v1):**
- **F1**: "X -sa/-se ne olur?" → forward, pozitif koşul
- **F2**: "X -mazsa/-mezse ne olur?" → forward, contrapositive
- **F3**: "X neden Y?" / "Y neden" → backward

**Kapsam-dışı (v2+):** yes/no doğrulama, genel graf sorgusu,
çok-hop zincir (Aşama 5), normal koşulsuz sorular (SPEAK değişmez).

### 13.2 Nasıl çalışıyor (reprodüksiyon için yeterli)

1. **Trigger parser** (`causal_speak_try`): prompt tokenize, koşul
   eki (`-sa/-se/-san/-sen/-arsa/-erse/-ırsa/...`) veya `neden` yakala.
2. **Negation marker** (`cau_has_neg_marker`): `m[a|e]z` substring
   aorist-negative yakalama. **Loop i=2'den başlar** (2-char kök
   "em" → "emmezse" için kritik).
3. **Canonical polarity**:
   `polarity = (cause_neg == effect_neg) ? +1 : -1`
4. **BFS + hop decay**: `forward_top/backward_top` bounded BFS (≤3),
   `conf × 0.7^(depth-1)`, polarite zincir boyunca **çarpılır**.
5. **Inverse canonicalization (surface)**:
   `effect_neg = cause_neg XOR (pol == -1)`
   → "olur" / "olmaz" seçimi.
6. **Short-circuit**: trigger + conf ≥ 0.30 → kausal cevap; aksi
   hâlde SPEAK normal akışı (trans/wpos/cooc boost).

### 13.3 Sonuç (4.5-D kapanış testi, 30 sorgu)

| Grup | Form | Skor |
|------|------|------|
| A | 10 F1 forward pozitif | **10/10** |
| B | 5 F2 forward negatif (contrapositive) | **5/5** |
| C | 10 F3 backward "çünkü" | **10/10** |
| D | 5 baseline-intact (trigger yok) | **5/5** |
| **Toplam** | | **30/30** |

**Analitik probe'lar (strongest-wins / noise-resistance):**
- `yemek neden pişer` → **çünkü ateş** (10+ in-edge, w=0.95 ev=13)
- `oynar neden` → **çünkü tilki** (7+ in-edge, w=0.60 ev=6)
- `içer neden` → **çünkü kuş** (8+ in-edge, w=0.65 ev=7)

**K3 polarity_collision sanity:** 3/3 ✅

**Baseline regresyon:** Test(500)=**58.2%**, v5.0 QA=**65.9%**
— entegrasyon öncesi ile bire bir.

### 13.4 Kazanım (neden önemli)

Contrapositive'i kimse öğretmedi. "erken kalkmazsan → geç olur"
cevabı, graf aritmetiğinden türedi: kanonik `polarity=+1`,
`cause_neg=1`, `effect_neg = 1 XOR 0 = 1` → "olmaz" beklenirdi
ama graftaki edge `kalk→erken` (geç pozitif ile rekabet),
polarite çarpımı ve inverse canonicalization otomatik "geç olur"
üretti. **Sistem cevap vermiyor — gerekçe üretiyor.**

### 13.5 Ne yapamıyor (şimdiden sabitlenmiş sınırlar)

- ❌ **Multi-hop zincir** (A→B→C): BFS zaten çok-hop yürür ama
  test setinde 1-hop edge'ler baskın. Sıra Aşama 5.
- ❌ **Conflict resolution**: K3 çelişkiyi **tespit** ediyor,
  **çözmüyor** (winner seçimi yok).
- ❌ **Why-chain**: "çünkü Y" sonrası "Y neden?" otomatik
  devam etmiyor (tek-adım).
- ❌ **Hypothesis injection**: causal cevap SPEAK hipotez havuzuyla
  yarışmıyor — short-circuit (ileride gerekirse açılır).
- ❌ **"değil"/"yok" partikül negasyonu**: yalnızca ek-bağlı `m[a|e]z`.

### 13.6 Resmi kapanış

- Organel eklendi: **SPEAK × Kausal Köprü** (Sinaptik bağlanma) → ✅
- Artefaktlar: `ribozom_causal.{h,c}` (speak_try + forward_top +
  backward_top), `ribozom_qa.h` (forward decl + hook),
  `_asama45_faz_{a,b,c,d}.sh` test setleri, `ribo31_causal.bin`
  snapshot.
- **Aşama 4.5 resmi kapanış saati: 19:xx.**

### 13.7 Causal Core v1 (stable) — resmi isim

Özellikler: single-hop reasoning ✔ / backward ✔ / negation ✔
/ polarity ✔ / strongest-cause ✔ / regression-safe ✔.

Checkpoint: bu noktadan önceki binary + dataset + config birlikte
**reprodüksiyon paketi**dir. Aşama 5 buradan dallanır.
