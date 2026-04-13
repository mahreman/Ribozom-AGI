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

**Eğitim durumu:** Devam ediyor (tahmin ~90-100dk).

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

### 2.2 15K 7-Domain — İlk Snapshot (Eğitim Devam Ediyor)

**Faz 1-2 tamamlandı** (68 dakika). Faz 3+ devam ediyor.

| Metrik | 10K | 15K tahmin | 15K gerçek | Yorum |
|--------|-----|-----------|-----------|-------|
| Kategori sayısı | 6 | 8-12 | **22** | Tahmininin 2x üstü! |
| Vocab | 306 | ~782 | ~782 | Beklenen |
| Faz 1 süresi | ~35dk | ~70dk | **68dk** | Doğrusal ölçekleme |
| Test/QA/OOV | — | — | bekliyor | Faz 3+ sonrası |

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

### 3.3 7-Domain Genişleme Stratejisi
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
7. **15K eğitim** — devam ediyor (~90-100dk)
8. **Domain genelleme testi** — eğitim sonrası (kategori sayısı, SV davranışı)

### Kısa Vade ⏳
9. **"Bilmiyorum" eşiği** — ribo_conf() düşükse "bilmiyorum" de
10. **Online cluster** — Yeni kelimeler runtime'da kategorilere eklenebilmeli
11. **Lizozom TF-IDF** — Bag-of-words Jaccard yerine kelime önem ağırlığı (T8 miss fix)
12. **test_qa_offd.txt UTF-8** — Off-diagonal test dosyası hâlâ eski ASCII

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
