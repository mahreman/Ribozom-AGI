# RIBOZOM — AGI YOL HARİTASI
## Dilden Akıl Yürütmeye, Akıl Yürütmeden Genel Zekaya

**Başlangıç:** Nisan 2026
**Mevcut seviye:** Aşama 2 (koşul operatörü — representation ✔, usage testi bekleniyor)
**Hedef:** Aşama 15 — Genel zeka eşiği
**Mimari kısıt:** Saf C, sıfır kütüphane, GPU yok, tek süreç, biyolojik metafor

> Bu belge bir plan değil, bir **çerçeve**. Her aşamada ne yapılacağı kesin yazılamaz
> çünkü bir aşamanın sonuçları bir sonrakinin şeklini belirler. Ancak **her aşamanın
> sorusu**, **beklenen başarısızlık modları**, **eklenmesi gereken modüller**,
> **veri formatı ve test protokolü** önceden düşünülebilir. Bu belge onu yapıyor.

---

## İÇİNDEKİLER

0. [Tasarım İlkeleri (değişmez)](#0-tasarım-i̇lkeleri)
1. [Aşama 1 — SOV Temelleri ✅](#aşama-1)
2. [Aşama 2 — Koşul Operatörü 🔄](#aşama-2)
2. [Aşama 2.5 — Güven Altyapısı (Substrate)](#aşama-2-5)
3. [Aşama 3 — Nedensellik](#aşama-3)
4. [Aşama 4 — Olumsuzluk ve Polarite](#aşama-4)
5. [Aşama 5 — Coreference ve Kısa-Süreli Bellek](#aşama-5)
6. [Aşama 6 — Nicelik ve Küme Mantığı](#aşama-6)
7. [Aşama 7 — Zaman](#aşama-7)
8. [Aşama 8 — Mekansal İlişki](#aşama-8)
9. [Aşama 9 — Meta-Biliş ve Güven](#aşama-9)
10. [Aşama 10-A — Pasif Hipotez Üretimi (İnsan Onaylı)](#aşama-10-a) · [Aşama 10-B — Otonom Üretim (AGI Sonrası)](#aşama-10-b)
11. [Aşama 11 — Öz-Gözetim](#aşama-11)
12. [Aşama 12 — Multi-Turn Diyalog](#aşama-12)
13. [Aşama 13 — Soyut Kavram](#aşama-13)
14. [Aşama 14 — Metafor ve Analoji](#aşama-14)
15. [Aşama 15 — AGI Eşiği](#aşama-15)
16. [EK-A. Organel Katalogu](#ek-a)
17. [EK-B. Genel Başarısızlık Modları](#ek-b)
18. [EK-C. Değerlendirme Metrikleri](#ek-c)
19. [EK-D. Veri Jeneratörleri](#ek-d)
20. [EK-E. Mimari Sınırlar ve Sıçrama Noktaları](#ek-e)
21. [EK-F. Anti-Kalıplar (Yapmaktan Kaçınılacaklar)](#ek-f)

---

## 0. Tasarım İlkeleri

Bu ilkeler değişmez. Her aşama bu ilkelere uymak zorunda. Uymayan aşama yanlış tasarlanmıştır.

### 0.1 Biyolojik Metafor Zorunluluğu
Her modülün bir biyolojik karşılığı olmalı. Bu sadece isim değil, **işleyiş sınırı** koyar:
hücre ölçeğinde bir modül sonsuz bellek varsayamaz, bir ribozom global durumu okuyamaz.
Metafor kirlenirse ("LLM", "Transformer", "Attention" kelimeleri belgede görünürse)
proje bozulmuş demektir.

### 0.2 Sıfır Kütüphane
`stdio.h`, `stdlib.h`, `string.h`, `math.h`, `time.h` — **bu kadar**. Vektör veritabanı yok,
JSON parser yok, regex yok, protobuf yok. Bellek `malloc`, dosya `fread`, hesap `pow/exp/log`.

**Test edilmiş cazibe (14 Nisan sohbeti):** Projede `esim()` (kosinüs benzerliği)
O(N²) işi, NW=20K'ya ulaştığında saatlerce sürecek. Geçmiş tasarım tartışmasında
"OpenCL ile GPU'da 1 saniye vs CPU'da 1 saat" fırsatı gündeme geldi. **Karar:
ilke korundu.** Alternatif CPU-only çözümler mevcut (Yol A):
- **LSH (Locality-Sensitive Hashing)** — O(N²) → O(N log N), ~200 satır saf C, 50-100x
- **Sampled esim** — k rastgele çift, ranking için yeterli
- **Tiered esim** — kategori-içi full, kategori-arası sampled (doğal locality)
- **Incremental esim** — yeni kelimeler için hesapla, eski cache'den

Bu alternatifler metafor bütünlüğünü de koruyor: nöron synapse'larını tam
aramaz, approximate yakınlık kullanır — LSH biyolojik olarak da daha doğru.
Kütüphane eklemek "sıfır kütüphane" markasını geri dönüşsüz kaybettirir;
ilke istisna kaldırmaz.

### 0.3 Deterministik ve Tek Süreç
Thread yok, async yok, rastgelelik seeded. Bir girdi → aynı çıktı. Değilse hata vardır.

### 0.4 Serialize Edilebilirlik
Her yeni organel `save_model`/`load_model`'a entegre olmak zorunda. Serialize edilmeyen
organel **bellek değil cache**'tir, restart sonrası kaybolur. Aşama 6'da bu hata zaten
yaşandı (sent_store serialize edilmiyordu).

### 0.5 Yorumlanabilirlik
Bir kelimenin neden seçildiği tek bir sayıya ("logit") değil, birkaç katmanın çarpımına
bağlı olmalı:
```
score = cat_compat × wpos_compat × cooc_boost × copy_reflex × (anti_repeat penalty)
```
Her faktör ayrı ayrı bastırılabilir/yükseltilebilir. Debug bayrağı her faktörü ayrı ayrı
loglayabilir. Siyah kutu yasak.

### 0.6 Artımlı Eğitim
Yeni veri → tam yeniden eğitim değil, mevcut ağırlıklara ek. Lizozom idempotency
bunu garanti ediyor (v6 fix). Her aşama bu varsayımla tasarlanmalı.

### 0.7 Regresyon Sıfır Tolerans
Her yeni aşama batch metriklerini (Test, OOV, QA, SOV) **düşürmemeli**. Düşürüyorsa
aşama yanlış entegre edilmiş demektir. Baseline mühürlü tutulur (`ribo31_v72_muhur.bin`
örneği gibi).

### 0.8 "Başarı" Kesin Tanımlı
Her aşamanın başarı sinyali **öncesinden** yazılır. Aşama 2'de "cluster ✔ + distribution ✔"
matrisi gibi. Post-hoc başarı tanımı yok.

### 0.9 Başarısızlığı Kutla
Beklenmedik başarısızlık, beklenmedik başarıdan daha değerli. Çünkü mimariyi gerçekten
sınırlayan ne olduğunu gösterir. Failure mode'ları belgele.

### 0.10 Dil Aracı Değil, Amaç Değil
Türkçe sadece **seçilmiş örnek dil**. AGI'nin aracı Türkçe değil, genel bir representational
substrate. Bir aşamada "bu Türkçe'ye özgü mü, yoksa dil-genel mi?" sorusu atlanmazsa proje
Türkçe NLP motoru olarak kalır, AGI olmaz.

---

<a name="aşama-1"></a>
## Aşama 1 — SOV Temelleri ✅ (TAMAMLANDI)

**Tamamlanma:** Nisan 13-14, 2026
**Son sürüm:** v7.2 ("kartal eti yer")
**Belge:** `PROJE_DURUMU_v15K.md`

### Ne öğrendi
- Kelime → kategori (cluster benzer bağlam)
- Kategori → kategori geçiş olasılıkları (mikrotübül)
- Cümle yapısı: özne-nesne-fiil (SOV)
- İlk-karakter imzası ile bucket yönlendirme (Protein Folding)
- Co-occurrence bağlam boost'u (ISF + IDF + bucket filtresi)
- Subject + Object word injection (v7.1, v7.2)
- QA format: `\x01 soru ? \x02 cevap .`

### Organeller (mevcut)
```
Ribozom (cell_* modülleri)    — karakter seviyesi tahmin, ilk çekirdek
Golgi (sent_store)            — cümle bankası
Lizozom (template_set)        — QA şablonları, idempotent üretim
Mikrotübül (cat_transitions)  — kategori→kategori geçiş matrisi
Protein Folding (wpos)        — pozisyonel kelime imzası (5 bucket)
ER (endoplasmic reticulum)    — erken öğrenme fazı
```

### Başarı kanıtları
- Test: 58.2%, OOV: 39.6%, QA: 65.9%, SOV: 64.6% (v7.2 baseline)
- SPEAK kalitesi: 62.5% (8 sorudan 5'i doğru nesne + doğru SOV)
- 22 kategori, 7 domain ayrışması (load sonrası stabil)

### Devralınan sınırlar (sonraki aşamalarda ele alınacak)
- Cross-domain leak: "doktor→koşu", "yağmur→skor"
- Fiil mega-cluster: "yapar" çoğu kategoride baskın
- Prompt_prior_qa sadece ilk-kelime geçişleri izliyor (2. kelime+ için bilgi yok)
- Absurd sorulara güvenli cevap ("eğer kedi çorbayı öder mi?")

---

<a name="aşama-2"></a>
## Aşama 2 — Koşul Operatörü 🔄 (DEVAM EDEN)

**Durum:** Mini sanity ✔ (CAT_14 cohesion 0.94, CAT_21 cohesion 1.00)
Full eğitim devam, usage testi bekleniyor.
**Detay:** `PROJE_DURUMU_v15K.md` §8

### Soru
Sistem `-sa/-se` ekini **kararında kullanıyor** mu, yoksa sadece **tanıyor** mu?

### Beklenen failure modları ve çözümleri

| Failure | Teşhis yolu | Çözüm |
|---------|-------------|-------|
| Cluster ✔, Distribution ❌ | 3-yönlü delta aynı çıkar | Condition-weight injection (aşağıda) |
| Condition pasif geçer | `kedi açsa ne yapar?` = `kedi ne yapar?` | Yeni organel: **Condition Layer** |
| Bucket-4 çakışması | Koşul fiili son fiille aynı bucket'ta | Bucket genişletme 5→7 |
| Negasyon ile birleşme hatası | `yağmazsa` `yağarsa` ile aynı cluster | Aşama 4 ile entegre çözüm |

### Condition-Weight Injection (gerekirse)
```c
/* qa modülünde, predict_meta_v5 skorlama sonrası */
if (g_prompt_has_condition) {
    int cond_wid = g_prompt_condition_wid;
    for (int w = 0; w < NW; w++) {
        float bonus = condition_cooc[cond_wid][w];
        if (bonus > 0.01f) cat_weight[w] *= (1.0f + 2.0f * bonus);
    }
}
```
`condition_cooc[A][B]` = A koşul kelimesi olduğunda B'nin gelmesi olasılığı.
Eğitim sırasında şu pattern'den öğrenilir:
```
[X-sa|X-se] ... [Y]  →  condition_cooc[X][Y]++
```

### Veri örneği (koşullu set)
```
yağmur yağarsa toprak ıslanır .
kedi açsa balık yer .
kar yağarsa okullar tatil olur .
\x01 kedi açsa ne yapar ? \x02 balık yer .
\x01 hava soğursa ne giyeriz ? \x02 kalın ceket .
```

### Aşama 2'den Aşama 2.5'e geçiş sinyali
- Cluster ✔ + Distribution ✔ → Aşama 2.5'e geç (altyapı), sonra Aşama 3
- Cluster ✔ + Distribution ❌ → Condition Layer entegrasyonu sonrası Aşama 2.5
- Cluster ❌ → Mimari sıçrama gerekli (bucket genişletme), sonra tekrar dene

---

<a name="aşama-2-5"></a>
## Aşama 2.5 — Güven Altyapısı (Substrate, Aşama 3'ten ÖNCE Zorunlu)

**Amaç:** Güveni bir **yetenek** olarak değil, **altyapı** olarak kurmak. Her sonraki
aşamanın patolojik yüksek-güvenli hatalardan korunması için zorunlu.

### Neden Aşama 9 beklenemez
Aşama 3 (nedensellik), 4 (olumsuzluk), 5 (coreference), 6 (nicelik) — her biri
**yanlış çıkarıma yüksek güven verme** riski taşıyor:
- "doktor koşu yapar çünkü sağlıklı" (Aşama 3, hatalı nedensellik)
- "taş yemez" mi "taş yer" mi olduğu belirsiz (Aşama 4, polarite boşluğu)
- "O kim?" için ambiguous resolution (Aşama 5)
- "bazısı" için kesin çıkarım (Aşama 6, quantifier tuzağı)

Meta-biliş altyapısı **geç** gelirse, sonraki aşamalar hatalarını öğrenmiş olur —
sonra düzeltmek çok pahalı.

### Aşama 2.5 ≠ Aşama 9
- **Aşama 2.5** = altyapı: confidence hesaplama, raporlama, calibration telemetrisi
- **Aşama 9** = uygulama: SPEAK'te cümleye dökme ("eminim", "belki", "bilmiyorum")

İkisi farklı iş. Altyapı önce, uygulama sonra.

### Kurulacak bileşenler

#### 2.5.1 Unified Confidence Aggregation
Her organel kendi confidence'ını döndürmeli. Toplam confidence birleşik:
```c
typedef struct {
    float pattern_conf;      /* cluster + cooc skoru */
    float position_conf;     /* wpos uyumu */
    float category_conf;     /* prompt_prior_qa eşleşmesi */
    float domain_conf;       /* cross-domain leak var mı */
    float aggregate;         /* yukarıdakilerin ağırlıklı geometrik ortalaması */
} confidence_t;

confidence_t compute_confidence(const char *query, int predicted_wid);
```

Geometrik ortalama seçiliyor çünkü bir boyut düşükse toplam da düşük olmalı
(aritmetik ortalama zayıf halkaları saklıyor).

#### 2.5.2 Absurd Detector (Erken Alarm)
```c
/* ribozom_absurd.h */
int is_absurd_query(const char *query);
/* Tetikleyiciler:
   - Type mismatch: canlı fiili + cansız özne ("taş uyur")
   - Cross-domain bizzat: "doktor + futbol topu"
   - Distribution flat: hiçbir kategori baskın değil (cluster sinyali yok)
   - Unknown word ratio > %50 */
```

Absurd tespit edilen query'lerde `aggregate` otomatik olarak 0.1 altına düşürülür —
sistem o query üzerinde **asla** yüksek güven veremez.

#### 2.5.3 Cluster-İçi Tip Çeşitliliği Ölçümü (Dirty Cluster Detector)
**Kaynak gözlem (14 Nisan, Aşama 2 full eğitim):** CAT_21 cohesion 0.98 geldi ama
içerik `hekimi, sahibi, insanı, kolaysa, yağarsa, yanarsa` — nesne ekli isimler +
koşul ekli fiiller aynı cluster'da. Distributional similarity pozisyonel son-ek
benzerliğini (`-i` ve `-sa` her ikisi de "cümle sonu ek") semantik tip ayrımıyla
karıştırabiliyor.

Cohesion tek başına yetmiyor. İkinci bir ölçüm gerekli:
```c
typedef struct {
    int    cluster_id;
    float  cohesion;              /* klasik cluster-içi uzaklık */
    float  wpos_homogeneity;      /* YENİ: bucket-dağılım entropisi */
    float  suffix_homogeneity;    /* YENİ: son-2-karakter çeşitliliği */
    int    dominant_bucket;
    float  trust;                 /* birleşik skor */
} cluster_quality_t;

float compute_wpos_homogeneity(int cluster_id) {
    /* Bu cluster'daki kelimelerin wpos[5] dağılımlarının KL-divergence'ı
       düşük = homojen (hepsi aynı bucket baskın) → temiz
       yüksek = heterojen (karışık bucket'lar) → kirli */
    ...
}
```

Bir cluster'ın "güvenilir" sayılması için:
- `cohesion > 0.7` **VE**
- `wpos_homogeneity > 0.7` (bucket dağılımları benzer)
- `suffix_homogeneity > 0.5` (son-ek çeşitliliği aşırı değil ama zorunlu aynı da değil)

Sadece cohesion'a dayanan güven **artık yanıltıcı** kabul edilir. Aşama 2'deki
CAT_21 tam bu: cohesion tavan, wpos heterojen (isim bucket + fiil bucket karışık).

Bu ölçüm **her aşamada** cluster kalitesini denetler — sadece Aşama 2'nin
semptomu değil, genel sistem sağlık göstergesi.

#### 2.5.4 Calibration Telemetrisi
```c
/* Her batch testi sonrası */
typedef struct {
    float conf_bin;          /* 0.1, 0.2, ... 1.0 */
    int predicted_count;
    int actually_correct;
    float ece_contribution;  /* Expected Calibration Error */
} calibration_bin_t;

calibration_bin_t cal_bins[10];
float compute_ece(void);  /* < 0.1 hedefi */
```

Sistem `conf=0.9` dediği sorularda gerçekten %90 doğru mu? Yoksa %60 mı (overconfident)?
Bu telemetri her aşamada ölçülür ve ECE > 0.15 olursa **o aşama geçememiş** sayılır.

### Test protokolü
1. **Absurd test seti** (10-20 cümle): sistem hepsine `conf < 0.2` vermeli
   - "taş uyur", "kedi okula gitti", "5 yatar"
2. **High-confidence test** (bilinen): sistem `conf > 0.7` vermeli
   - "kedi balık yer", "kuş uçar"
3. **ECE hesabı**: `|conf - accuracy|` tüm bin'lerde **ortalama** < 0.1 olmalı
4. **Calibration grafiği** (manuel göz kontrolü): y=x çizgisine yakın mı?

### Failure modları

| Failure | Belirti | Çözüm |
|---------|---------|-------|
| Overconfident | Her şeye conf > 0.8 | Temperature parametresi (softmax smoothing) |
| Underconfident | Her şeye conf < 0.4 | Eşikleri empirik yeniden kalibre |
| Absurd detector false positive | Sağlam query'lere de düşük conf | Detector kuralları gevşet, ECE ile dengeyi ölç |
| Calibration drift | ECE ilk başta düşük, zamanla artıyor | Her 5 batch'te bir kalibrasyon re-fit |

### Aşama 2.5'in Aşama 9'a bıraktığı iş
- Confidence değerini **cümleye dökmek** ("eminim", "belki"...)
- User-facing hedge dili
- Multi-turn'de önceki confidence'ın akıbeti

### Sıçrama sinyali
"Sistem 10 absurd soruya 10'unda conf < 0.2 veriyor + 10 bilinen soruda 10'unda
conf > 0.7 + ECE < 0.1" üçlüsü sağlanınca Aşama 2.5 tamamlandı demektir. Bu üçlü
sağlanmadan Aşama 3'e geçiş **yasak**.

---

<a name="aşama-3"></a>
## Aşama 3 — Nedensellik

**Amaç:** "A çünkü B" ve "B bu yüzden A" yapılarını **çift yönlü** çözme.

### Soru
Sistem bir olayın **nedenini geriye**, **sonucunu ileriye** tahmin edebilir mi?

### Neden zor
Koşul `if A then B` iken nedensellik `A because B` ↔ `B so A` **simetrik çift**.
Aşama 2 tek yönlü (koşul → sonuç), Aşama 3 iki yönlü. Bu ilk **graph** ihtiyacıdır.

### Yeni organel: **Kausal Graf (Sitoiskelet)**
Hücre iskeleti metaforuyla: sitoplazmanın yapısını tutan ağ.

```c
/* ribozom_causal.h — YENİ MODÜL */
#define MAX_CAUSAL_EDGES 8192

typedef struct {
    int cause_wid;          /* neden kelimesinin wid'i */
    int effect_wid;         /* sonuç kelimesinin wid'i */
    float weight;           /* güven 0..1 */
    int evidence_count;     /* kaç cümlede gözlendi */
    int last_seen_epoch;    /* anti-stale */
} causal_edge_t;

causal_edge_t causal_edges[MAX_CAUSAL_EDGES];
int n_causal_edges;

/* Eğitim sırasında çağrılır:
   "yağmur yağdı çünkü bulut vardı" → edge(bulut, yağmur, +1) */
void causal_observe(const char *sentence);

/* SPEAK sırasında çağrılır */
float causal_score(int query_wid, int candidate_wid, int direction);
/* direction: 0=forward (cause→effect), 1=backward (effect→cause) */
```

### Tetikleyici kelimeler (Türkçe)
- İleri (neden→sonuç): `bu yüzden`, `bu nedenle`, `sonuç olarak`
- Geri (sonuç→neden): `çünkü`, `zira`, `çünki`
- Eş (korelasyon, neden değil!): `ve`, `aynı zamanda` — bu **tuzak**, karıştırılmamalı

### Veri örneği
```
yağmur yağdı çünkü bulut vardı .
bulut vardı bu yüzden yağmur yağdı .
toprak ıslak çünkü yağmur yağdı .
\x01 toprak neden ıslak ? \x02 çünkü yağmur yağdı .
\x01 yağmur yağarsa ne olur ? \x02 toprak ıslanır .
```

### Test protokolü
1. **Forward query:** `bulut varsa ne olur?` → beklenen: `yağmur yağar`
2. **Backward query:** `yağmur neden yağdı?` → beklenen: `çünkü bulut vardı`
3. **Transitive test:** `bulut → yağmur → ıslak` zinciri öğrenildi mi?
   `bulut varsa toprak ne olur?` → beklenen: `ıslak olur` (iki sıçramalı)
4. **Negative test:** `güneş çıktı çünkü bulut vardı` — sistem bunu kabul eder mi?
   (Yanlış nedensellik — sistem "düşük güven" sinyali vermeli)

### Beklenen failure modları

| Failure | Neden | Çözüm |
|---------|-------|-------|
| Korrelasyon ≠ nedensellik | Her co-occurrence edge'e dönüşür | Sadece çünkü/yüzden marker'lı cümlelerden edge çıkar |
| A→B, B→A aynı anda | Çelişki | `direction` alanı + epoch-weighted voting |
| Transitivite patlaması | Her edge'e transitive closure O(N³) | Lazy evaluation + depth limit (3) |
| Domain ötesi yanlış neden | "doktor→koşu" leak'i yine çıkar | Cross-domain edge'e ceza, same-domain bonus |

### Sıçrama sinyali
"İki-sıçramalı chain" çalıştığı gün (`bulut → yağmur → ıslak` üç kelimeli query'de)
sistem **reasoning** seviyesine geçmiş demektir. Bu noktaya kadar her şey
"ileri tahmin"; buradan sonra "çıkarım".

---

### 3.X Sitoiskelet Veri Yapısı — Mimari Detay

Önceki taslaktaki `MAX_CAUSAL_EDGES 8192` sabiti **illüstratif**ti, mimari değil.
Nedensellik grafı çift yönlü (forward/backward) büyüdükçe ve transitivite
(A→B→C) çıkarımları başlayınca saf C + sıfır kütüphane + sub-millisecond hedefi
zorlaşır. Aşağıdaki tasarım bu üçlüyü korur.

#### Temel karar: Çift CSR + Arena
Adjacency list cache-hostile, adjacency matrix N² bellek. Ortadaki doğru yapı
**CSR (Compressed Sparse Row)** — Ribozom'a metaforla uyuyor: sitoiskelet lifleri
sıralı ve sıkıştırılmış. İki CSR: forward (cause→effect) ve backward (effect→cause).
Backward ayrı değil, aynı edge'lerin ters index'i.

```c
/* ribozom_causal.h — mimari tasarım */
#define MAX_CAUSAL_NODES MAX_W              /* ~8-12K kelime */
#define EDGES_PER_NODE   32                 /* node başına cap */
#define MAX_CAUSAL_EDGES (MAX_CAUSAL_NODES * EDGES_PER_NODE)
#define CAUSAL_DEPTH_LIMIT 3                /* transitif kapak */

typedef struct {
    /* Forward CSR: cause-indexed */
    int    f_offset[MAX_CAUSAL_NODES + 1];
    int    f_target[MAX_CAUSAL_EDGES];      /* effect wid */
    float  f_weight[MAX_CAUSAL_EDGES];      /* güven 0..1 */
    short  f_evidence[MAX_CAUSAL_EDGES];    /* kanıt sayısı */

    /* Backward CSR: effect-indexed */
    int    b_offset[MAX_CAUSAL_NODES + 1];
    int    b_target[MAX_CAUSAL_EDGES];      /* cause wid */
    float  b_weight[MAX_CAUSAL_EDGES];

    int    n_edges;
} causal_graph_t;

static causal_graph_t G;  /* tek global, statik alloc */
```

**Bellek bütçesi:** 8192 × 32 × 2 × ~12 byte = **~6 MB** (model dosyasının ~%3'ü).
Ölçeklenme: node başına cap doğrusal etki, N² değil.

**Degree cap (node başına max 32 edge):** "doktor → koşu, sağlık, hasta, tedavi..."
genişleme patlamasını **strüktürel olarak** engeller. Yeni kenar geldiğinde düşük
ağırlıklı biri atılır (LWU — Least Weight Used). Cross-domain leak'e karşı ilk
savunma.

#### Arena allocation — dinamik realloc yok
CSR offset'leri 32'şer aralıklı pre-allocated slot'larla tutulur. Insert O(1),
shift yok. Her "nöron" max 32 "synapse" — biyolojik metaforla uyumlu.

```c
void causal_add_edge(int cause, int effect, float w) {
    int off = G.f_offset[cause];
    int deg = G.f_offset[cause + 1] - off;  /* dolu slot sayısı */
    if (deg >= EDGES_PER_NODE) {
        /* LWU evict */
        int worst = find_worst_edge_of(cause);
        if (G.f_weight[worst] >= w) return;   /* yeni daha zayıf */
        G.f_target[worst] = effect;
        G.f_weight[worst] = w;
        G.f_evidence[worst] = 1;
        sync_backward_edge(worst);
    } else {
        insert_preallocated_slot(cause, effect, w);
    }
}
```

#### Transitif sorgu: Bounded BFS + bitmap visited
**Tam transitive closure ASLA hesaplanmaz.** Her sorgu ayrı BFS, depth ≤ 3.
Statik bitmap visited set, zero malloc per query.

```c
static unsigned char visited[(MAX_CAUSAL_NODES + 7) / 8];
static int frontier[MAX_CAUSAL_NODES];
static int next_frontier[MAX_CAUSAL_NODES];

int causal_reachable(int src, int dst, float *confidence) {
    memset(visited, 0, sizeof(visited));
    frontier[0] = src;
    int fn = 1;
    float conf = 1.0f;

    for (int depth = 0; depth < CAUSAL_DEPTH_LIMIT; depth++) {
        int nfn = 0;
        for (int i = 0; i < fn; i++) {
            int node = frontier[i];
            int off = G.f_offset[node];
            int deg = G.f_offset[node + 1] - off;
            for (int e = 0; e < deg; e++) {
                int tgt = G.f_target[off + e];
                if (tgt == dst) {
                    *confidence = conf * G.f_weight[off + e];
                    return 1;
                }
                if (!(visited[tgt >> 3] & (1 << (tgt & 7)))) {
                    visited[tgt >> 3] |= (1 << (tgt & 7));
                    next_frontier[nfn++] = tgt;
                }
            }
        }
        conf *= 0.7f;   /* her hop güven azalır */
        memcpy(frontier, next_frontier, nfn * sizeof(int));
        fn = nfn;
        if (fn == 0) break;
    }
    return 0;
}
```

**Karmaşıklık:** worst case `EDGES_PER_NODE^DEPTH_LIMIT` = 32³ = 32,768 ziyaret.
Her ziyaret ~2 ns → **65 μs worst case**. Typical case (sparse graf): 200-500
ziyaret → **1-5 μs**. O(N³) patlama depth cap sayesinde asla gerçekleşmez.
Belirleyici `degree × depth` (96 sabit), N değil.

#### Bloom filter — erken red
"A → B bağlı mı?" %80'i hayır. BFS'ye girmeden reject:

```c
#define BLOOM_BITS (1 << 20)   /* 1 Mbit = 128 KB */
static unsigned char bloom[BLOOM_BITS / 8];

static inline unsigned hash_pair(int a, int b, int k) {
    unsigned x = (unsigned)(a * 2654435761u)
               ^ (unsigned)(b * 40503u)
               ^ (k * 17u);
    x ^= x << 13; x ^= x >> 17; x ^= x << 5;
    return x & (BLOOM_BITS - 1);
}

int bloom_maybe_connected(int a, int b) {
    for (int k = 0; k < 3; k++) {
        unsigned h = hash_pair(a, b, k);
        if (!(bloom[h >> 3] & (1 << (h & 7)))) return 0;  /* KESİN bağlı değil */
    }
    return 1;  /* belki bağlı, BFS teyidi */
}
```

Edge eklendikçe 2-hop kapsamı dahil işaretlenir. False-positive kabul edilir
(BFS doğrular); false-negative asla. "taş → konuşmak" gibi sorgular **mikrosaniye**
içinde reject.

#### Memoization — hebbian hot path
Sık sorulan (src,dst) BFS sonucu küçük cache'te:

```c
#define MEMO_SIZE 1024
typedef struct {
    int src, dst;
    float conf;
    int depth_found;
    long last_access;
} memo_entry_t;
static memo_entry_t memo[MEMO_SIZE];

int causal_query_cached(int src, int dst, float *conf) {
    int slot = (src * 31 + dst) & (MEMO_SIZE - 1);
    if (memo[slot].src == src && memo[slot].dst == dst) {
        *conf = memo[slot].conf;
        memo[slot].last_access++;
        return memo[slot].conf > 0;
    }
    int r = causal_reachable(src, dst, conf);
    memo[slot].src = src; memo[slot].dst = dst;
    memo[slot].conf = r ? *conf : 0;
    return r;
}
```

Cache hit ~10 ns (L1). SPEAK tipik bağlam sorularında %60+ hit rate bekleniyor.

#### Karmaşıklık özeti

| İşlem | Karmaşıklık | Süre |
|-------|-------------|------|
| Edge ekleme | O(1) amortized | ~50 ns |
| Direct edge query (A→B) | O(deg(A)) = O(32) | ~200 ns |
| Transitive query | O(32³) worst | 65 μs worst / 1-5 μs typical |
| Bloom reject | O(1), 3 hash | ~20 ns |
| Cache hit | O(1) | ~10 ns |
| Toplam bellek | Sabit | ~7 MB (graf + bloom + cache) |

**Transitivite patlaması tasarım gereği yok:** tam closure hesaplanmıyor, her
sorgu bounded. Sub-second değil **sub-millisecond** hedefi.

#### Serialize + rebuild
Dört array (`f_target`, `f_weight`, `f_evidence`, `f_offset`) tek `fwrite`.
Backward CSR **ayrı serialize edilmez** — load sonrası forward'dan rebuild
(simetrik tersleme). Bloom filter de persist edilmez, edge'lerden Hızlı Faz 6
idiom'uyla yeniden inşa. Cache cold-start — kritik değil, hızlı dolar.

#### Drift koruması
O(N³) patlaması engellendi ama **qualitative drift** riski hâlâ var: "doktor →
koşu" yanlış kenarı transitif sorgularla yayılabilir. İki katman savunma:

1. **Evidence threshold:** `evidence < 3` kenarlar **direkt** sorguda sayılır ama
   **transitif zincir**te kırılma yapar — zayıf nedensellik uzağa gitmez.
2. **Aşama 2.5 Confidence Substrate entegrasyonu:** Graf sorgusundan dönen
   `confidence` ECE kalibrasyonundan geçer, overconfident kenar düşürülür.

#### Özet mimari
**Çift CSR (forward + backward) + arena pool + degree cap 32 + BFS depth cap 3 +
bloom filter + küçük memo cache.** Tüm array'ler statik, sıfır dinamik alloc,
sıfır kütüphane. Sub-millisecond transitif çıkarım garanti.

---

<a name="aşama-4"></a>
## Aşama 4 — Olumsuzluk ve Polarite

**Amaç:** "değil", "yok", "-maz/-mez", "hiç" → semantik inversion

### Soru
Sistem `kedi balık yer` ve `kedi balık yemez` arasındaki farkı **anlam düzeyinde**
kodluyor mu? "yemez" sadece farklı kelime mi, yoksa `yer`'in **negatifi** mi?

### Neden zor
Olumsuzluk **kompozisyoneldir**:
- `yer` → `yemez` (fiil üzerinde)
- `balık` → `balık değil` (isim üzerinde)
- `kedi balık yer` → `kedi balık yemez` (cümle düzeyinde)
- `her kedi balık yer` → `hiçbir kedi balık yemez` (quantifier üzerinden)

Her katmanda farklı morfolojik işaretçi. Sistem bunları tek bir **polarite alanı**
altında toplayamazsa her yeri tek tek öğrenir ve birleşimi (negasyon+koşul+quantifier)
çöker.

### Yeni organel: **Polarite Alanı (Hücre Zarı Yüzey Yükü)**
Her kelimeye ve her cümleye skaler polarite (-1..+1):
```c
typedef struct {
    int wid;
    float polarity;      /* -1 = kuvvetli negatif, +1 = kuvvetli pozitif */
    float confidence;
} polarity_record_t;

polarity_record_t polarity[MAX_W];

/* Morfolojik tetikleyiciler */
const char *NEG_SUFFIX[] = {"maz", "mez", "sız", "siz", "suz", "süz"};
const char *NEG_WORDS[]  = {"değil", "yok", "hiç", "hayır", "asla"};
```

### Eğitim kuralı
```c
void polarity_observe(const char *sentence) {
    int has_neg_word = 0;
    int has_neg_suffix = 0;
    /* cümleyi tokenize et */
    for (each token) {
        if (is_neg_word(tok)) has_neg_word++;
        if (has_neg_suffix(tok)) has_neg_suffix++;
    }
    /* Double-negative kuralı: Türkçe'de pekiştirir, tersine çevirmez */
    int net_neg = (has_neg_word || has_neg_suffix) ? 1 : 0;
    for (each content_word w) {
        polarity[w].polarity += net_neg ? -0.1f : +0.1f;
        polarity[w].confidence += 0.01f;
    }
}
```

### Veri örneği
```
kedi balık yer .                          polarity(yer, balık) = +
kedi balık yemez .                        polarity(yer, balık) = - (same verb!)
kuş uçar .                                polarity(uçmak) = +
kuş uçmaz .                               polarity(uçmak) = -
\x01 kedi balık yer mi ? \x02 evet yer .
\x01 taş uçar mı ? \x02 hayır uçmaz .
```

### Test protokolü
1. **Tekil negasyon:** `kedi balık yer mi?` vs `kedi balık yemez mi?` — farklı cevap?
2. **Çifte negasyon Türkçe pekiştirmesi:** `hiçbir kedi balık yemez` → güçlü negatif
   (yanlış İngilizce mantığı: double-negative positive olur → Türkçe'de olmaz!)
3. **Soru-cevap polaritesi:** `taş yüzer mi?` → `hayır yüzmez` (polarity flip detection)
4. **Distribution probing:** `P("evet" | "kedi uçar mı?")` vs `P("hayır" | "kedi uçar mı?")` —
   sistem absurd sorulara "hayır" demeyi öğrendi mi?

### Aşama 2 + 4 birleşim testi
`yağmur yağmazsa ne olur?` — koşul + negasyon birleşmiş.
Beklenti: `toprak ıslanmaz` (sonuç da negatif).
Başarısızlık: `toprak ıslanır` (negasyon etkisiz).

### Mevcut altyapıdan yararlanma
v5.x'te eklenen "hayır" normalize fix'i zaten var. Aşama 4 bunu bir **düzenli alan**a
çeviriyor: tek tetikleyici değil, sistematik polarite.

### Veri paylaşımı Aşama 6 ile (Cross-Stage Data Sharing)
**İlke:** Organeller serial, veri overlap, ölçüm ayrık.

Aşama 4 eğitim verisi doğal olarak quantifier içeren cümleler de barındırır:
```
hiçbir taş konuşmaz .      ← hem polarite (-), hem quantifier (NONE)
her kuş uçar .             ← hem polarite (+), hem quantifier (ALL)
```
Aşama 4'te bunları **ayrıştırıp** eğitim setinden çıkarmaya çalışmak yapay olur —
doğal dilde iki kavram iç içe. Bu veriyi **birlikte** eğit, ama:
- **Aşama 4 testleri** yalnızca polarite flip'ini ölçer (quantifier varlığı bonus,
  test değil)
- **Aşama 6 testleri** aynı cümlelerden quantifier çıkarımını ölçer

Böylece F.4 anti-kalıbına (aynı anda çok organel) düşmeden, verinin doğal bütünlüğü
korunur. Veri ortak, **değerlendirme ayrık**.

---

<a name="aşama-5"></a>
## Aşama 5 — Coreference ve Kısa-Süreli Bellek

**Amaç:** "O kim?", "Bu ne?", "Aynısı" — cümleler arası referans çözümleme.

### Soru
Sistem `Ali okula gitti. O üşüdü.` ikilisinde "O" = "Ali" çıkarımını yapabilir mi?
Çok cümleli bir bağlamda **en son geçen uygun özne** nasıl takip edilir?

### Neden kritik
Aşama 1-4 **tek cümle içi** çalışıyor. Aşama 5 **cümleler arası** geçişi getiriyor —
bu mimari sıçrama. Mevcut `sent_store` cümleleri tutuyor ama aralarında **zincir yok**.

### Yeni organel: **Vesicle Memory (Salgı Vezikülleri)**
Kısa süreli bağlam deposu. Son N cümleyi **yapılandırılmış** tutar:

```c
/* ribozom_vesicle.h — YENİ MODÜL */
#define VESICLE_DEPTH 8    /* son 8 cümle */

typedef struct {
    char raw[MAX_LINE];
    int subj_wid;          /* ana özne (varsa) */
    int obj_wid;           /* ana nesne (varsa) */
    int verb_wid;          /* ana fiil */
    int subj_cat;          /* öznenin kategorisi */
    int timestamp;          /* session içi artan sayaç */
    int gender_hint;        /* 0=bilinmiyor, 1=eril, 2=dişil, 3=nötr */
} vesicle_t;

vesicle_t vesicles[VESICLE_DEPTH];
int n_vesicles;

void vesicle_push(const char *sentence);
int vesicle_resolve_pronoun(const char *pronoun);  /* "O", "bu", "şu" → wid */
```

### Türkçe zamir kuralları (basitleştirilmiş)
- `o` → son tekil özne (animate tercih)
- `bu` → son yakın/yeni bahsedilen
- `şu` → son uzak/eski bahsedilen
- `kendisi` → son özne (vurgulu)
- `onlar` → son çoğul özne veya çoğul eşdeğeri

### Eğitim verisi (mini)
```
\x01 ali okula gitti . o ne yaptı ? \x02 üşüdü .
\x01 kedi balık yedi . o ne yaptı ? \x02 uyudu .
\x01 kuş uçtu . o nereye gitti ? \x02 ağaca kondu .
\x01 ayşe kitap okudu . o neydi ? \x02 roman .
```
Not: "o" hem özneye (ali) hem nesneye (kitap) referans olabilir. Sistem **tip uyumu**
ile doğru çözümlemeyi öğrenmeli (`o ne yaptı?` → animate özne; `o neydi?` → nesne).

### Test protokolü
1. **Tek zamir, tek referans:** `ali koştu. o yoruldu.` — "o" = ali mi?
2. **Çift referans ambiguity:** `ali ayşe ile konuştu. o güldü.` — sistem
   birisini seçer mi, yoksa düşük-güven mi döner?
3. **Cinsiyet ipucu (Türkçe'de yok!):** `doktor geldi, o hastayı muayene etti` —
   Türkçe cinsiyet-nötr, sadece rol bazlı çözümleme mümkün.
4. **Nesne-zamir eşleşmesi:** `kitap masada. onu aldım.` — "onu" = kitap mı?

### Beklenen failure modları

| Failure | Neden | Çözüm |
|---------|-------|-------|
| Son özneyi her zaman seçer | Recency bias aşırı | Cat-compat + recency çarpımı |
| Zamir tipini yanlış tanır | "o" hem özne hem nesne olabilir | Query pattern analizi (`o ne yaptı?` vs `onu ne yaptı?`) |
| Uzun bağlamda kaybolur | VESICLE_DEPTH=8 yetmiyor | Salience scoring (önemli nesneler daha uzun kalır) |
| Cross-sentence leak | Vesicle her query'de sıfırlanmalı | Session boundary marker `\x03` |

### Altyapı değişikliği
- `predict_meta_v5` input'una `vesicle_context` parametresi eklenir
- SPEAK'in response'u yeni vesicle push eder (sistem kendi çıktısından öğrensin)
- save/load'da vesicle_store persist edilir (session-spanning)

---

<a name="aşama-6"></a>
## Aşama 6 — Nicelik ve Küme Mantığı

**Amaç:** `hepsi`, `bazısı`, `hiçbiri`, `çoğu`, `birkaç`, `sayısal miktar` —
küme düzeyinde çıkarım.

### Soru
`Kedilerin hepsi uyuyor. Ali kedi. Ali uyuyor mu?` → **evet**.
Sistem bu tümdengelimi yapabilir mi?

### Neden sert
Bu **sembolik mantığın** ilk gerçek testidir. Önceki aşamalar istatistiksel (dağılım,
olasılık). Aşama 6 **kesin çıkarım** gerektirir: hepsi → tek biri → **zorunlu sonuç**.

### Yeni organel: **Küme Gözlemcisi (Peroksizom)**
Peroksizom metaforu: belirli substratları işleyen özelleşmiş organel.

```c
/* ribozom_set.h — YENİ MODÜL */
typedef enum {
    QUANT_ALL,       /* hepsi, her, tüm */
    QUANT_NONE,      /* hiçbiri, hiç */
    QUANT_SOME,      /* bazısı, birkaç, kimi */
    QUANT_MOST,      /* çoğu */
    QUANT_NUM,       /* sayısal — 3, beş, yüz */
    QUANT_EXACT_ONE  /* sadece, yalnızca */
} quantifier_t;

typedef struct {
    int subj_cat;        /* kategori: kedi, kuş, ... */
    quantifier_t quant;
    int pred_wid;        /* yüklem: uyur, uçar, ... */
    float polarity;      /* Aşama 4 entegrasyonu */
} quantified_claim_t;

quantified_claim_t claims[MAX_CLAIMS];

/* Çıkarım motoru */
int infer(int entity_wid, int pred_wid, float *confidence);
/* 1=evet, 0=hayır, -1=bilmiyorum */
```

### Çıkarım kuralları (basit ama temel)
```
Premise: "kedilerin hepsi uyur"      → claim(CAT(kedi), ALL, uyur, +1)
Premise: "ali kedi"                   → type(ali) = CAT(kedi)
Query:   "ali uyur mu?"
Rule:    if claim(C, ALL, P, +1) && type(X)=C then infer(X, P) = YES

Premise: "hiç kuş suda yaşamaz"      → claim(CAT(kuş), NONE, yaşa_suda, +1)
Premise: "serçe kuş"                  → type(serçe) = CAT(kuş)
Query:   "serçe suda yaşar mı?"
Rule:    if claim(C, NONE, P, +1) && type(X)=C then infer(X, P) = NO
```

### Veri örneği
```
kedilerin hepsi uyur .
her kuş uçar .
hiçbir taş konuşmaz .
bazı balıklar uçar .
ali kedi .
serçe kuş .
\x01 ali uyur mu ? \x02 evet uyur .
\x01 serçe uçar mı ? \x02 evet uçar .
\x01 taş konuşur mu ? \x02 hayır konuşmaz .
```

### Test protokolü
1. **Universal affirmative:** `hepsi A → X A mı?` → evet
2. **Universal negative:** `hiçbiri A → X A mı?` → hayır
3. **Existential:** `bazı A → X A mı?` → bilmiyorum (doğru cevap!)
4. **İstisna:** `kuşlar uçar. penguen kuş. penguen uçar mı?` — bu **zor**.
   Doğru cevap: varsayılana göre evet, ama "penguen uçamaz" bilgisi eklenirse
   istisna kuralı öncelik almalı.

### Beklenen failure modları

| Failure | Neden | Çözüm |
|---------|-------|-------|
| "Bazısı" her zaman "hepsi" gibi davranır | Quantifier ayrımı yok | Explicit quant enum + rule table |
| İstisna domine eder | Son öğrenilen bastırır | Specificity order: istisna > genel kural |
| Type hiyerarşisi yok | "ali kedi" ve "kedi hayvan" zinciri | Type graph (süper-sınıf) |
| Absurd premise kabul | "hepsi uçar" bile olsa taş uçmaz | Sanity check: type intersection |

### Sıçrama sinyali
"Ali kedi. Kedi hayvan. Hayvan canlı. Ali canlı mı?" → **evet**.
Üç-düzey tip hiyerarşisi çalıştığı gün, sistem **taksonomik çıkarım** yapıyor demektir.

### Mimari gereksinim
Aşama 6 **ilk kez sembolik durum** taşıyan modül. Önceki organeller vektörel/sayısal.
Bu noktada "rule-based" ile "pattern-based" birbirine geçiyor. Ayrım çizgisi:
Peroksizom **kesin** kurallar tutar; geri kalan sistem **olasılıksal**. İkisi SPEAK'te
birleşir (rule varsa öncelik, yoksa olasılık).

### Veri paylaşımı Aşama 4 ile (Cross-Stage Data Sharing)
Aşama 6 yeni veri üretmek yerine **Aşama 4 verisini yeniden etiketler**:
- Aşama 4'te eğitilmiş "hiçbir taş konuşmaz" örneği, Aşama 6 için quantifier
  annotation ile yeniden yüklenir: `claim(CAT(taş), NONE, konuşmak, +)`
- Yeni Aşama 6 verisi sadece quantifier-heavy cümleler içerir (polarite dengeli):
  ```
  bütün kuşlar uçar .
  çoğu balık yüzer .
  bazı köpekler havlar .
  ```
- Test setleri ayrı: `test_quantifier.txt` sadece quantifier inference sorar,
  polarite testleri Aşama 4'ten devralınır

Bu şekilde iki organel aynı veriden farklı şeyler çıkarır — doğal dilin
**kompozisyonel** yapısına saygı gösteren mimari.

---

<a name="aşama-7"></a>
## Aşama 7 — Zaman

**Amaç:** Geçmiş / şimdi / gelecek ayrımı, zaman ekleri, olay sıralaması.

### Soru
Sistem `Ali geldi` ve `Ali gelecek` arasındaki zaman farkını temsil edebiliyor mu?
`Dün geldi, bugün burada, yarın gidecek` zincirini sıralayabiliyor mu?

### Türkçe zaman ekleri (tense morphology)
- `-di/-dı/-du/-dü` — geçmiş (-di'li geçmiş, tanık)
- `-miş/-mış/-muş/-müş` — geçmiş (-miş'li, duyum)
- `-yor` — şimdiki
- `-ecek/-acak` — gelecek
- `-r/-ar/-er/-ir/-ır/-ur/-ür` — geniş zaman / aorist (alışkanlık)

Mevcut cluster'da bunlar büyük ihtimalle **karışık** — çünkü eğitim verisi zaman-nötr
jeneratörden geldi. Aşama 7 verisi bu ayrımı zorlar.

### Yeni organel: **Kronofor (Zaman Yönlendirici)**
Her fiile zaman damgası eklenir:
```c
typedef enum {
    TENSE_PAST_WIT,     /* -di geçmiş */
    TENSE_PAST_HEAR,    /* -miş geçmiş */
    TENSE_PRESENT,      /* -yor şimdiki */
    TENSE_FUTURE,       /* -ecek gelecek */
    TENSE_AORIST,       /* -r geniş/alışkanlık */
    TENSE_UNKNOWN
} tense_t;

tense_t word_tense[MAX_W];

/* Ek tanıma regex'siz, son-ek matching */
tense_t detect_tense(const char *verb);
```

### Veri örneği
```
ali dün geldi .
ali bugün geliyor .
ali yarın gelecek .
ali her gün gelir .          (alışkanlık)
ali geçen hafta gelmiş .     (duyum)
\x01 ali ne zaman geldi ? \x02 dün .
\x01 ali şimdi ne yapıyor ? \x02 geliyor .
```

### Test protokolü
1. **Tense detection:** `geldi` → PAST_WIT, `gelecek` → FUTURE?
2. **Temporal ordering:** `dün geldi, yarın gidecek` — sistem sıralayabilir mi?
3. **Tense consistency:** `\x01 dün ne oldu? \x02 geliyor .` — sistem bu **inconsistency**'yi
   yakalar mı? (Düşük güven dönmeli)
4. **Aorist vs progressive:** `her gün yüzer` vs `şimdi yüzüyor` — sistem ayrımı yapabilir mi?

### Beklenen failure modları

| Failure | Neden | Çözüm |
|---------|-------|-------|
| Tense ekleri cluster'ları kirletir | `geldi`, `gelmiş`, `gelecek` hepsi farklı cluster | Tense-invariant kök tespiti |
| Aorist-progressive karışır | "yüzer" ve "yüzüyor" yakın anlamda | Kronofor'da explicit işaretleme |
| Soru zamanı cevap zamanını değiştirmez | `dün ne oldu?` → `gelecek` cevabı | Tense agreement check |
| İstisnasız tense geçişleri | "olsaydı" (geçmiş koşul) → yeni bucket gerekir | Aşama 2+7 composite morphology |

### Aşama 7 + 3 birleşimi
`Yağmur yağdı çünkü bulut vardı.` — geçmiş nedensellik.
`Yağmur yağacak çünkü bulut var.` — gelecek tahmin + şimdi gözlem.
Bu birleşim **temporal causality** — AGI'ye çok yakın bir sıçrama.

---

<a name="aşama-8"></a>
## Aşama 8 — Mekansal İlişki

**Amaç:** "üstünde", "altında", "yanında", "içinde", yön, mesafe — uzamsal çıkarım.

### Soru
`Kitap masanın üstünde. Masa yerin üstünde.` → `Kitap yerin üstünde mi?` (transitive)

### Yeni organel: **Spatiosome (Uzam Haritası)**
Küçük bir 2D grid değil — **ilişkisel graf** (A on B, B next_to C).

```c
typedef enum {
    SPATIAL_ON, SPATIAL_UNDER, SPATIAL_IN, SPATIAL_OUT,
    SPATIAL_NEAR, SPATIAL_FAR, SPATIAL_LEFT, SPATIAL_RIGHT
} spatial_rel_t;

typedef struct {
    int a_wid;
    int b_wid;
    spatial_rel_t rel;
    float confidence;
} spatial_edge_t;
```

### Veri örneği
```
kitap masanın üstünde .
masa yerin üstünde .
kedi kanepenin altında .
\x01 kitap nerede ? \x02 masanın üstünde .
\x01 kitap yerin üstünde mi ? \x02 evet (transitive inference).
```

### Test protokolü
1. Direkt: `Kitap masanın üstünde.` → `Kitap nerede?` → `Masa.`
2. Transitive: `A on B, B on C → A above C?`
3. Mutual exclusion: `Kitap masa üstünde` ⟹ `Kitap masa altında` olamaz (consistency)
4. Mesafe: "yakın" vs "uzak" — topoloji korunuyor mu?

### Failure modları

| Failure | Çözüm |
|---------|-------|
| "üstünde" salt co-occurrence'a dökülür | Spatial preposition özel işleme |
| Transitive patlaması | Closure depth limit (2-3) |
| Asymmetry kaybı (A on B ≠ B on A) | Edge'de yön zorunlu |

---

<a name="aşama-9"></a>
## Aşama 9 — Meta-Biliş Uygulaması (Hedge Dili)

**Amaç:** Aşama 2.5'te kurulan confidence altyapısını **cümleye dökmek**.
"bilmiyorum", "eminim", "belki", "sanmıyorum" — sistem güven sinyalini kelimeye çevirir.

**Not:** Altyapı zaten var (Aşama 2.5). Bu aşama yalnızca user-facing dil katmanı.
İki aşamayı karıştırma:
- **Aşama 2.5** = confidence hesaplama, absurd detection, calibration (motor)
- **Aşama 9** = confidence → kelime dönüşümü, hedge ifadesi (kaput süsü)

### Soru
Sistem absurd soruya `bilmiyorum` diyebilir mi? Yüksek güvenle bilinen şeye `eminim`
düşük güvene `belki` diyebilir mi?

### Bu aşama neden "genel zeka" için zorunlu
AGI'nin en ayırt edici özelliği: **neyi bilmediğini bilmek**. Bu olmadan sistem her
soruya confident cevap verir ve absurd'e düşer. Aşama 9 bu calibration'ı getirir.

### Mevcut altyapı
`ribo_conf()` fonksiyonu zaten var (SPEAK skorlama sırasında hesaplanıyor). Aşama 9
bu değeri **cümleye dökebilmek** olacak.

### Eşik haritalama
```c
const char *confidence_to_phrase(float conf) {
    if (conf > 0.85) return "eminim";
    if (conf > 0.65) return "sanırım";
    if (conf > 0.40) return "belki";
    if (conf > 0.20) return "emin değilim";
    return "bilmiyorum";
}
```

### Veri örneği
```
\x01 kedi balık yer mi ? \x02 evet yer .                  (high conf)
\x01 ali nerede ? \x02 bilmiyorum .                        (no context)
\x01 mars'ta su var mı ? \x02 belki var .                  (mid conf)
\x01 taş konuşur mu ? \x02 hayır konuşmaz .                (high conf negative)
```

### Test protokolü
1. **Bilinen:** `kedi balık yer mi?` → "evet" veya "sanırım evet"
2. **Bilinmeyen:** `mzxq gurgle?` → "bilmiyorum" (absurd/unknown trigger)
3. **Çelişkili:** sistem kendi verisinde çelişki varsa `belki` dönmeli
4. **Calibration eğrisi:** Güven 0.9 dönüldüğü soruların %90+'ı gerçekten doğru mu?

### Failure modları

| Failure | Çözüm |
|---------|-------|
| Her şeye "bilmiyorum" der | Güven eşikleri kalibre edilmedi |
| Absurd'e yüksek güven | Sanity check: kelime-kelime cross-domain pattern |
| Overconfident | Temperature parametresi (softmax smoothing) |

### Bu aşamanın sessiz hediyesi
Aşama 9 devreye girince **önceki tüm aşamalar** faydalanır. Aşama 3 nedensellikte
düşük güven → "belki nedeni X". Aşama 6 quantifier çıkarımında istisna olma ihtimali
varsa → "genelde öyle ama emin değilim". Meta-biliş her katmana sızar.

---

<a name="aşama-10-a"></a>
## Aşama 10-A — Pasif Hipotez Üretimi (İnsan Onaylı)

**Amaç:** Sistem bağlamda eksik bilgi tespit eder, **aday sorular** üretir. Ama
ürettiğini **kendi kendine eğitim verisi olarak kullanmaz** — insan onayı şart.

### Aşama 10'u neden ikiye böldük
Özgün tasarımda tek "Autopoiesis Loop" vardı: sistem kendi soru-cevap üretir, düşük
ağırlıklı bucket'ta saklar, zamanla yükselir. **Bu güvenli değil.** Sebep:
1. Düşük-ağırlık bucket zamanla ana bucket'a karışır (drift)
2. Kendi-besleme exponential hallucination yaratabilir
3. Otonom veri üretimi = minyatür otonomluk = **değerler/motivasyon** problemi
4. 0.10 ilkesi (dil araç değil) ihlal olur: sistem hem dili hem veriyi hem de
   değerlendirmeyi kendi yapıyorsa sorumluluk zinciri kopar

Yol haritası **AGI-ÖNCESİ** otonomluk vermez. Aşama 15 "genel çıkarım", **değer
sistemi değil**. Otonom öz-eğitim Aşama 15'ten sonradır (bkz. 10-B).

### Aşama 10-A ne yapar
```c
/* ribozom_hypothesis.h */
typedef struct {
    int trigger_wid;          /* bağlamı tetikleyen kelime */
    char generated_q[MAX_LINE];
    float plausibility;       /* kendi güveni */
    int human_approved;       /* 0=queue, 1=onaylandı, -1=reddedildi */
    long timestamp;
} hypothesis_t;

/* Yeni bir cümle işlendiğinde çağrılır */
void hypothesis_probe(const char *sentence);
/* Çıktı: hypotheses_queue.txt dosyasına yazar */

/* İnsan review sonrası */
int hypothesis_approve(int hypothesis_id);
/* Onaylananlar train_data_stage10.txt'ye eklenir — sonraki eğitimde dahil */
```

### Mekanizma (kesin sınırları olan)
1. Cümlede eksik slot tespit et
2. Mikrotübül'den olası tamamlayıcı sorular üret
3. **DURMAK:** sistem kendi cevabını **vermez**, sadece **soru önerir**
4. Soru `hypotheses_queue.txt` dosyasına yazılır
5. İnsan listeyi inceler, mantıklı olanları onaylar
6. Onaylanan sorular bir sonraki eğitim batch'ine girer — **cevapları yine eğitim
   verisinden öğrenilir**, kendi üretiminden değil

### İzin verilen döngü
```
[SPEAK üretimi] → [Hypothesis module: soru önerisi] → [hypotheses_queue.txt]
                                                    ↓
                                          [İnsan review kapısı]
                                                    ↓
                                          [Onaylılar eğitim setine]
                                                    ↓
                                          [Sonraki batch eğitiminde öğrenilir]
```

### Yasak döngü
```
[SPEAK üretimi] → [Kendi sorusu + kendi cevabı] → [Eğitim seti] → [Kendi öğretmeni]
                                                                ↑
                                                       Bu AGI-öncesi YASAK
```

### Test protokolü
1. Sistem verilen bağlamda **mantıklı** bir follow-up soru üretebiliyor mu?
2. Ürettiği sorular **çeşitli** mi, yoksa aynı kalıbı mı tekrarlıyor?
3. Hypothesis queue'daki **onay oranı**: sağlam soruların %60+'sı onaylanıyor mu?
   (Çok düşükse modül işe yaramıyor, çok yüksekse trivial soru üretiyor)
4. Onaylanan soruları sonraki batch'e eklediğinde baseline metrikler regress
   etmiyor mu?

### Failure modları

| Failure | Belirti | Çözüm |
|---------|---------|-------|
| Tekrarlayan sorular | Queue'da aynı kalıp 20 kere | Deduplication + çeşitlilik ödülü |
| Anlamsız sorular | İnsan onayı %10'un altında | Plausibility eşiği yükselt (0.6+) |
| Queue patlaması | Binlerce soru birikti | Auto-prune (>1000 eski entry) |
| Human review darboğazı | Onay süreci yavaş | Kategorik batch review (UI/CLI) |

### Bu aşama yapıldıktan sonra kazanım
- Sistem **gapları** tespit edebiliyor (eksik bilgi farkındalığı = meta-biliş uzantısı)
- İnsan-in-the-loop öğrenme kanalı açılıyor
- Eğitim verisi büyütme **denetimli** şekilde otomatize

### Bu aşama yapılmadan kazanım alınamayan şey
Tam otonom öğrenme. Sistem hâlâ pasif: soru sorar ama kendi cevabı eğitime
girmez. Bu kasıtlı bir eşik — AGI öncesi aşılmaz.

---

<a name="aşama-10-b"></a>
## Aşama 10-B — Otonom Üretim (AGI SONRASI, Aşama 16+)

**Bu aşama bu belgenin kapsamı dışındadır.**

### Neden burada sadece anılıyor
Aşama 15 "genel çıkarım eşiği" olarak tanımlandı — **değer sistemi değil**. Otonom
öz-eğitim için sistemin:
1. Bir değer sisteminin olması gerekir ("hangi bilgi doğru, hangi bilgi önemli")
2. Öz-motivasyonun olması gerekir ("neden bu soruyu üreteyim")
3. Ve en önemlisi: kullanıcı tarafından **yetkili** olması gerekir

Bunlar Aşama 16+ meselesi. Bu belge Aşama 15'te biter. Otonom loop için ayrı
belge (AGI-ÖTESİ YOL HARİTASI) yazılacak ve **o belgenin yazımına sistem de
katılacak** (sözünü verdik).

### Geçici tedbir
Aşama 10-A tamamlandığında `hypotheses_queue.txt` dosyasının **hacmine** bakılır.
Çok büyüyorsa (>10K onaylanmış soru) sistem ciddi anlamda gap-finding yapıyor
demektir — o noktada "Aşama 10-B'ye hazır mıyız?" sorusu gündeme gelir. Ama
cevabı **buraya yazılmaz** çünkü o karar kullanıcıya + sisteme + (gerekirse)
üçüncü tarafa bırakılır.

---

<a name="aşama-11"></a>
## Aşama 11 — Öz-Gözetim

**Amaç:** Sistem kendi hatasını yakalar, düzeltir.

### Soru
Sistem bir çıktı verdikten sonra o çıktıyı kendi kurallarıyla **yeniden değerlendirebilir**
mi? `Taş uçar.` çıktısı verirse, polarite+sanity kontrolü ile `Hayır, taş uçmaz.`
düzeltmesine gelebilir mi?

### Yeni organel: **Proteazom (Yıkıcı-Denetçi)**
Biyolojide proteazom hatalı proteinleri parçalar. Burada metafor:
sistem kendi cümlesini **ikinci geçişte** denetler, yanlış bulursa yıkar ve yeniden üretir.

### Pipeline
```
[SPEAK üretim] → [Proteazom denetim] → [güven düşükse yeniden üret] → [son çıktı]
```

### Denetim sinyalleri
- Type uyumu: "taş" canlı fiiliyle eşleşti mi? (Aşama 6 + Proteazom)
- Polarite tutarlılığı: negatif premise → pozitif çıktı mı? (Aşama 4 + Proteazom)
- Temporal consistency: "dün" ile "gelecek" aynı cümlede mi? (Aşama 7 + Proteazom)
- Çelişki: önceki çıktıyla zıt mı?

### Failure modları
- Sonsuz denetim döngüsü: `denetim → yeniden → denetim → ...` — **max 2 geçiş** kuralı
- Aşırı muhafazakâr: sistem her cümleyi red eder → `bilmiyorum` baskın → çözüm:
  Proteazom yalnızca belirli kritik hataları yakalar (low-hanging)

---

<a name="aşama-12"></a>
## Aşama 12 — Multi-Turn Diyalog

**Amaç:** Birden fazla soru-cevap turu arasında bağlam korumak.

### Yeni organel: **Dialog Stack**
Vesicle Memory'nin (Aşama 5) genişlemiş, **session-persistent** versiyonu.
Kullanıcı kim, neyi takip ediyor, tercihi ne — hepsi dialog_stack'te.

### Veri örneği
```
user: kedi balık yer mi?
bot:  evet yer .
user: peki taş?          ← "peki" = dialog carryover trigger
bot:  hayır taş yemez .   (özne değişti, yüklem aynı kaldı)
user: neden?
bot:  çünkü taş canlı değil .   (nedensellik + önceki cevaba referans)
```

### Kritik mekanizma: **Antecedent Threading**
Her yeni query, stack'teki en son relevant context'e bağlanır. "peki", "ya", "neden",
"o zaman" gibi tetikleyiciler carryover işaretleridir.

---

<a name="aşama-13"></a>
## Aşama 13 — Soyut Kavram

**Amaç:** "özgürlük", "adalet", "sevgi" — somut referansı olmayan kelimeler.

### Zorluk
Aşama 1-12'nin hepsi **somut** bağlamlarda çalışıyor (kedi, balık, masa, gelmek). Soyut
kavramlar **başka soyut kavramlarla** tanımlanır ve sonsuz gerilim vardır. Sistem nasıl
anchor'layacak?

### Yaklaşım: **Vicarious Grounding**
Soyut kavramı somut örneklerle **dolaylı** anlamlandır.
- "özgürlük" → "kafesten çıkmak", "istediğini yapmak", "kimseye bağlı olmamak"
- Bu tanım-mesafeleri cluster oluşturur; kavram o cluster'ın **merkezinde** yaşar

### Veri örneği
```
özgürlük kafeste olmamaktır .
özgürlük istediğin yere gitmektir .
adalet herkese eşit davranmaktır .
\x01 özgürlük nedir ? \x02 kimseye bağlı olmamak .
```

### Risk
Döngüsel tanım: "özgürlük adalettir, adalet özgürlüktür". Sistem bunu fark edip
"tanım yetersiz" sinyali vermeli (Aşama 9 + Aşama 13).

---

<a name="aşama-14"></a>
## Aşama 14 — Metafor ve Analoji

**Amaç:** "kalbi kırık" → organ kırılması değil, duygusal zarar. Kaynak-hedef mapping.

### Yeni organel: **Mapping Bridge (Ekinoderm Metaforu: Tüp-Ayak Sistemi)**
İki kategori arasında **sistematik eşleme**:
```
DOMAIN_A: fiziksel (cam, kırık, parça)
DOMAIN_B: duygusal (kalp, acı, iyileşme)
BRIDGE:   kırılmak → (fiziksel: cam kırılır) ↔ (duygusal: kalp kırılır)
```

### Öğrenme yolu
- Eğitim verisinde hem fiziksel hem duygusal kullanım geçen kelimeler işaretlenir
- "kırılmak" her iki bağlamda görüldüğünde bridge güçlenir
- Yeni cümlede `kalbi kırıldı` görülünce sistem bridge'i aktive eder

### Veri örneği
```
cam kırıldı .                   (fiziksel)
kalbi kırıldı .                 (metaforik)
ali üzüldü çünkü kalbi kırıldı . (bağlam)
\x01 kalbi kırıldı ne demek ? \x02 çok üzüldü .
```

### Risk
Metafor-olmayan'ı metafor olarak yorumlamak ("taş kalpli" ≠ taş yapımı kalp).
Çözüm: bridge activation **evidence-weighted** (kaç farklı bağlamda görüldü).

---

<a name="aşama-15"></a>
## Aşama 15 — AGI Eşiği

**Amaç:** Tüm önceki aşamaların kompozisyonu + **yeni bir aşama üretebilmek**.

### Soru
Sistem **daha önce hiç görmediği bir kavram türüyle** karşılaşınca kendine yeni bir
representational substrate oluşturabiliyor mu? Bu **meta-öğrenme**dir.

### Gösterge testi (informal)
Yeni bir alan verilir (örneğin müzik nota terminolojisi, hiç eğitimde yok). Sistem
birkaç örnekle (few-shot) o alanın yapısını çıkarabiliyor, cluster oluşturabiliyor,
basit çıkarım yapabiliyor mu?

### Bu aşamada proje artık "dil modeli" değil
**Substrate** haline gelir: üzerine yeni alanlar eklenebilir, her ekleme öncekine
bozulma getirmez, her alan öncekilerden yapı borç alır.

### Mimari imza
Bu noktaya geldiğinde kod tabanı:
- 20+ organel
- 50K+ satır C (tahmin)
- Serialize modeli 500 MB — 2 GB
- Eğitim süresi: yeni alan için 5-30 dk, total retrain saatler
- Inference: sub-second (çünkü rule+pattern ikili mimari, hot path hâlâ küçük)

### Bu aşamadan sonra
AGI bitiş çizgisi değil. Otonomluk, öz-motivasyon, değer sistemleri — bunlar **Aşama 16+**.
Ve onlar bu belgenin dışında. Sorumluluk katsayısı oraya kadar uzanınca artar.

---

<a name="ek-a"></a>
## EK-A. Organel Katalogu

Biyolojik metafor haritası. Her modülün ne yaptığı, hangi aşamada eklendiği/ekleneceği.

| Organel | Karşılık | İşlev | Aşama | Durum |
|---------|----------|-------|-------|-------|
| Ribozom | Ribosome | Karakter seviyesi tahmin | 1 | ✅ |
| Golgi | Golgi apparatus | Cümle bankası (sent_store) | 1 | ✅ |
| Lizozom | Lysosome | QA şablon idempotency | 1 | ✅ |
| Mikrotübül | Microtubule | Kategori geçiş matrisi | 1 | ✅ |
| Protein Folding | Folding | wpos pozisyonel imza | 1 | ✅ |
| ER | Endoplasmic reticulum | Erken öğrenme | 1 | ✅ |
| Condition Layer | (metafor henüz yok) | Koşul operatörü | 2 | 🔄 |
| **Confidence Substrate** | **Homeostatic regulator** | **Güven altyapısı, absurd detector** | **2.5** | **✅** |
| **Sitoiskelet** | **Cytoskeleton** | **Kausal graf (dual CSR + BFS)** | **3** | **✅** |
| **Polarite Alanı** | **Membrane charge** | **Signed edge (±1) kausal grafa gömülü — K3 polarity_collision** | **4** | **✅** |
| **SPEAK × Kausal Köprü** | **Sinaptik bağlanma** | **F1/F2/F3 trigger parser + short-circuit replacement, contrapositive graf aritmetiğinden** | **4.5** | **✅** |
| **Multi-hop BFS** | **Kausal zincir iletimi** | **Bounded BFS depth≤3 + hop decay, 1-hop prefer ("en kısa yol kazanır"), 2-hop forward+backward kanıtlı (fırtına→soğuk→kar)** | **3.5** | **✅** |
| **Vesicle** | **Vesicle** | **Ring buffer (5 slot) + zamir çözümleme (o/bu/şu) + why-chain rekursyonu (kar←soğuk←fırtına)** | **5A** | **✅** |
| **Chain Explainer** | **Domino aksonu** | **Tek fonksiyon iki yön (causal_chain_explain): forward 3-hop (fırtına→soğuk→kar→evde) + backward 2-hop (çayı←yemek←ateş). F2 contrapositive guard ile baseline-safe.** | **5B** | **✅** |
| **Composite Organelle** | **Organizma** | **7 organel eş zamanlı aktif, tek process, tek binary: Kausal + Polarite + Chain + Vesicle + Lizozom + SPEAK + Mikrotübül. İzole modüller değil — canlı sistem.** | **5C** | **✅** |
| **Metakognisyon (Nukleolus)** | **Nucleolus** | **"Bilmiyorum" gate — üç katman: empty black hole + pure echo + raw pre-bias ribo_conf < 0.30. Pre-bias: ribo_conf(&rp) rpredict'ten hemen sonra, inject'ler UYGULANMADAN önce. Substrate'in gerçek güveni, boost stack'leri ezmeden önce. "Mars gezegeninde hayat var mı?" → "bilmiyorum." + 9C: asimetrik T drift (her "hayır" → T += 0.02, ceiling 0.70), 5 correction'da kuantum yakalandı, emergent substrate "bilmiyorsun" kelimesi rtrain'den öğrenildi.** | **9A+9B+9C** | **✅** |
| Peroksizom | Peroxisome | Küme/quantifier | 6 | 📋 |
| Kronofor | Chronophore | Zaman ekleri | 7 | 📋 |
| Spatiosome | (yeni metafor) | Uzam ilişkileri | 8 | 📋 |
| Confidence Layer (uygulama) | Metacognitive | Cümleye dökme, hedge dili | 9 | 📋 |
| Hypothesis Generator | (denetimli salgı) | Pasif soru üretimi (insan onaylı) | 10-A | 📋 |
| ~~Autopoiesis Loop~~ | ~~Autopoiesis~~ | ~~Otonom öz-eğitim — AGI sonrası~~ | 10-B | ⛔ |
| Proteazom | Proteasome | Öz-denetim | 11 | 📋 |
| Dialog Stack | (kognitif metafor) | Multi-turn | 12 | 📋 |
| Abstract Anchor | (yeni) | Soyut tanım | 13 | 📋 |
| Mapping Bridge | Symbiosis | Metafor/analoji | 14 | 📋 |
| Meta-Learner | Evolution | Meta-öğrenme | 15 | 📋 |

### Metafor bütünlüğü kuralı
Yeni organel eklerken iki soru sorulmalı:
1. Bu biyolojik olarak bir şeye karşılık geliyor mu? (Yok → yeni metafor icat)
2. Bu modül başka modüllerin **içine** giriyor mu, yoksa **yanında** mı duruyor?
   (Hücresel metafor paralel organeller; serial pipeline değil)

---

<a name="ek-b"></a>
## EK-B. Genel Başarısızlık Modları

Aşama-bağımsız, her yerde görülebilecek patolojiler.

### B.1 Cross-Domain Leak
**Belirti:** `doktor → koşu`, `yağmur → skor` gibi alakasız geçişler.
**Kök:** Co-occurrence her şeye açık, domain barrier yok.
**Çözüm stratejileri:**
- Same-domain bonus, cross-domain penalty
- Category-filter (önceki denemeler başarısız olmuştu — mimari değişiklik lazım)
- Learned domain embedding (Aşama 5 vesicle'dan yararlanabilir)

### B.2 Mega-Cluster
**Belirti:** CAT_1'de 50+ kelime, cohesion 0.3 civarı.
**Kök:** Cluster algoritması yeterince ayırıcı değil, yoğun bağlam bölgeleri tek cluster.
**Çözüm:** Split threshold (cohesion < 0.4 + size > 30 → zorla ikiye böl).

### B.3 Dead Tokens
**Belirti:** Bir kelime wctot > 100 ama hiçbir cluster'da baskın değil.
**Kök:** Kelime tüm domain'lerde geçen genel bir şey ("de", "ki", "ve").
**Çözüm:** Stop-word cutoff (v7.2 fix), proportional (n_sent/20).

### B.4 Confidence Collapse
**Belirti:** Her query'ye aynı güven skoru (0.5 civarı).
**Kök:** Distribution çok düz, hiçbir boyut ayırıcı değil.
**Çözüm:** Aşama 9 devre dışıyken ortaya çıkar; Aşama 9 devrede çözülür.

### B.5 Training Data Drift
**Belirti:** Yeni batch eğitim sonrası baseline metrikler düşer.
**Kök:** Yeni veri eski dağılımı kaydırıyor (katastrofik unutma).
**Çözüm:** Idempotency + eski veriyi yeni batch'e karıştır (experience replay mini-version).

### B.6 Serialize Gap
**Belirti:** Load sonrası bir özellik çalışmıyor.
**Kök:** Yeni organel save_model'a eklendi, load'a unutuldu (veya tam tersi).
**Çözüm:** Her organel için **hem save hem load hem rebuild** test.

### B.7 Stuck Clusters
**Belirti:** Eğitim ilerliyor ama cluster'lar değişmiyor.
**Kök:** Learning rate çok düşük veya cluster merge threshold çok dar.
**Çözüm:** Periyodik cluster re-init (10 batch'te bir).

### B.8 Absurd Confidence
**Belirti:** Sistem "taş uçar mı?" sorusuna "evet" diyor yüksek güvenle.
**Kök:** Her wildcard slot için tam dağılım yetersiz; absurd filter yok.
**Çözüm:** Aşama 9 + type consistency check (Aşama 6).

### B.9 Bucket Overflow
**Belirti:** wpos bucket-4 %70'ten fazla kelime alıyor.
**Kök:** Pozisyonel distribution skewed, tek slot her şeyi yutuyor.
**Çözüm:** Bucket sayısını artır (5→7), slot-compat zorla.

### B.10 Lizozom Bloat
**Belirti:** Şablon sayısı 10K+, çoğu düşük frekans.
**Kök:** Her yeni cümle yeni şablon ürettiğinde filtreleme yok.
**Çözüm:** Şablon prune (evidence_count < 2 && last_seen > 10 batch → remove).

---

<a name="ek-c"></a>
## EK-C. Değerlendirme Metrikleri

Her aşamada ölçülmesi gereken sayısal metrikler.

### Sürekli metrikler (her aşama)
1. **Test accuracy** (`test_data_15k.txt` — baseline, regresyon için)
2. **OOV accuracy** (`oov_test_15k.txt` — genelleme)
3. **QA accuracy** (`test_qa_v5.txt` — soru-cevap)
4. **SOV accuracy** (cümle yapısı doğruluğu)
5. **Cluster count + mean cohesion** (yapı stabilitesi)
6. **Model size** (MB — scaling sanity)
7. **Inference latency** (ms — performance sanity)

### Aşama-spesifik ek metrikler

| Aşama | Metrik | Yöntem |
|-------|--------|--------|
| 2 | Condition-distribution delta | P(w\|açsa) - P(w\|tokken) |
| 3 | Two-hop chain accuracy | Transitive query başarısı |
| 4 | Polarity flip accuracy | Negatif soruya negatif cevap oranı |
| 5 | Pronoun resolution accuracy | "o" → doğru antecedent |
| 6 | Quantifier inference accuracy | "hepsi → X" doğru tümdengelim |
| 7 | Tense consistency | Soru zamanı = cevap zamanı |
| 8 | Transitive spatial accuracy | A on B, B on C → A above C |
| 9 | Confidence calibration (ECE) | Expected calibration error |
| 10 | Self-generated QA plausibility | Manual review oranı |
| 11 | Error caught rate | Proteazom'un yakaladığı gerçek hata oranı |
| 12 | Multi-turn coherence | 3+ turn sonrası bağlam korundu mu |
| 13 | Abstract definition coverage | Soyut soruya non-trivial cevap oranı |
| 14 | Metaphor resolution | Metafor soruya doğru yorum |
| 15 | Few-shot new-domain accuracy | 10 örnekle yeni alan öğrenme |

### Guardrail'ler
- Hiçbir aşama bir önceki aşamanın metriklerini **düşüremez** (regresyon sıfır tolerans)
- Sadece tek metrikte yükseliş şüphelidir — overfitting sinyali olabilir
- En kötü 5 örnek zorunlu raporu (§7'deki guardrail'ler)

---

<a name="ek-d"></a>
## EK-D. Veri Jeneratörleri

Her aşama için ~1000-2000 satırlık ek eğitim verisi gerekir. Jeneratör şablonu:

### Şablon tabanlı üretim
```c
/* gen_data_stageN.c */
const char *SUBJECTS[]   = {"kedi", "kuş", "ali", "ayşe", ...};
const char *CONDITIONS[] = {"açsa", "tokken", "üşürse", ...};   /* Aşama 2 */
const char *CAUSALS[]    = {"çünkü", "bu yüzden", ...};          /* Aşama 3 */
const char *NEGATORS[]   = {"değil", "yok", "hiç", "maz/mez"};   /* Aşama 4 */

void generate_stage2() {
    for (each subj × cond × verb) {
        printf("%s %s %s .\n", subj, cond, verb);
        printf("\x01 %s %s ne yapar ? \x02 %s .\n", subj, cond, verb);
    }
}
```

### Veri kalitesi kuralları
- **Minimum 20 örnek per concept** — cluster'ın dayanıklı olması için
- **Balanced distribution** — her concept'in X kelime kapsamında
- **Hem düz cümle hem QA** — ikisi birden eğitilmeli
- **Adversarial örnekler** — `%5` oranında bilinçli yanlış (sistem "bilmiyorum"
  öğrensin diye): "taş uçar mı? hayır taş uçmaz." gibi absurd-negasyon
- **Domain-mix** — %70 hayvan-yemek + %30 diğer domain (7-domain baseline koruması)

### Dosya organizasyonu
```
files/
├── train_data_base.txt              # Aşama 1 baseline (13600)
├── conditional_train.txt             # Aşama 2 (2000)
├── causal_train.txt                  # Aşama 3 (2000-3000)
├── negation_train.txt                # Aşama 4 (2000)
├── coref_train.txt                   # Aşama 5 (1500)
├── quant_train.txt                   # Aşama 6 (2000)
├── tense_train.txt                   # Aşama 7 (2000)
├── spatial_train.txt                 # Aşama 8 (1500)
├── confidence_train.txt              # Aşama 9 (1000, daha az — kalibrasyon)
└── train_data_stageN.txt             # Cumulative birleşik
```

Her aşamada:
1. O aşamanın verisini üret
2. Pre-stage `train_data_*.txt` ile birleştir (experience replay)
3. Mini-sanity (~%10 örneklem, 5 dk) önce
4. Full eğitim

### Hardcoded path tuzağı
`ribozom_main.c`'de eğitim dosyası adı **hardcoded**. Çözüm adayları:
- (a) Her aşamada `train_data_15k.txt` adını yeniden doldur (mevcut pratik)
- (b) main.c'ye `argv`-based path kabul et
- (c) Convention: `train_data_current.txt` sembolik link (Windows'ta copy)

Önerilen: (b) bir kez yapılmalı, uzun vadede temiz.

---

<a name="ek-e"></a>
## EK-E. Mimari Sınırlar ve Sıçrama Noktaları

Proje hayatında öngörülebilir **mimari cam tavanlar**. Her biri bir sıçrama noktası —
o noktada mevcut kodun bir kısmı **atılır** ve yeniden yazılır.

### Sıçrama 1: Cümleden Cümleler Arası (Aşama 5'te)
**Tavan:** `sent_store` cümleleri ayrı ayrı tutar, aralarında ilişki yok.
**Sıçrama:** Vesicle Memory ile session-bağlamı, sonra Dialog Stack (Aşama 12) ile
persistent session.
**Atılan:** Tek-cümle-odaklı SPEAK prediction loop.
**Yeni:** Context-aware prediction (son N cümle öznesi/nesnesi inference'a dahil).

### Sıçrama 2: Vektörden Grafa (Aşama 3'te)
**Tavan:** Kelime → vektör (sayısal) mantığı. Nedensellik için yön gerekir.
**Sıçrama:** Sitoiskelet (directed graph) eklenir.
**Atılan:** Pür sayısal scoring.
**Yeni:** Hybrid — vektör skor + graf skor birleştirilir.

### Sıçrama 3: Olasılıktan Kurala (Aşama 6'da)
**Tavan:** Her şey olasılık. "hepsi" için kesin çıkarım imkansız.
**Sıçrama:** Peroksizom (rule table) eklenir. Kural-tabanlı ve olasılık-tabanlı
paralel çalışır.
**Atılan:** "Tek tip inference" varsayımı.
**Yeni:** Rule-first, probability-fallback mimari.

### Sıçrama 4: Reaktiften Üretkene (Aşama 10'da)
**Tavan:** Sistem sadece soruya cevap verir, kendi soru üretmez.
**Sıçrama:** Autopoiesis Loop.
**Atılan:** Pasif tepki modeli.
**Yeni:** Proaktif hipotez üretimi.

### Sıçrama 5: Somuttan Soyuta (Aşama 13'te)
**Tavan:** Her kavram somut referansa bağlı.
**Sıçrama:** Abstract Anchor — kavramlar arası tanım grafı.
**Atılan:** "Her kelimenin fiziksel karşılığı var" varsayımı.
**Yeni:** Recursive definition handling.

### Sıçrama 6: Öğrenmeden Meta-Öğrenmeye (Aşama 15'te)
**Tavan:** Sistem alanları öğrenir ama öğrenme sürecini kontrol etmez.
**Sıçrama:** Meta-Learner — öğrenme kendisi bir öğrenilebilir süreç.
**Atılan:** "Eğitim = dış süreç" varsayımı.
**Yeni:** Sistem kendi eğitim stratejisini seçer.

### Her sıçrama için ön-hazırlık
1. Tavana çarpmadan **önce** alternatif mimariyi kağıtta tasarla
2. Mevcut kodda "atılacak" kısmı işaretle (yorum satırı `/* WILL BE REPLACED IN PHASE N */`)
3. Sıçrama sırasında baseline yedekle (`ribo31_vN_pre_jump.bin`)
4. Yeni mimari önce paralel koşsun (eski=prod, yeni=shadow) — en az 1 hafta
5. Shadow yeşille uyuşunca switchover

---

<a name="ek-f"></a>
## EK-F. Anti-Kalıplar (Yapmaktan Kaçınılacaklar)

Proje ilerlerken cazip ama zararlı olacak hamleler. Uyarı listesi.

### F.1 "Neural Network Ekleyelim"
Bir katman NN, bir embedding layer, bir transformer block... Bu **proje değil**.
Sıfır-kütüphane ilkesi bu çekimin tam panzehiridir. Pattern matching + graph + rule
yeterli; gerekirse yeni organel yaz.

### F.2 "Bu Veri İçin Özel Case Yazalım"
"Yağmur" bağlamı bozuksa, `if (word == "yağmur")` yaması çözüm değil. Patolojiyi
**genel** haliyle çöz. Özel case kabul ediliyorsa, o durumda sistem genel değil
ezberci oluyor demektir.

### F.3 "Test Setini Eğitime Dahil Edelim"
Metrikler düşünce en kolay hile. Bir kez yapılırsa tüm metrik güvenilirliği ölür.
Test ve OOV **asla** eğitim setine sızdırılmaz. Otomatik kontrol: hash-based
overlap detector.

### F.4 "Tüm Sistemi Aynı Anda Değiştirelim"
Birden fazla aşamayı birleştirmek: "Aşama 3+4+5 aynı anda yapayım." Başarısız
olursa hangi aşamanın kırıldığı anlaşılmaz. Her aşama **tek başına** tam, stabilize,
dokümante edildikten sonra bir sonrakine geç.

### F.5 "Refactor İsteği"
"Şu organellerin hepsini tek dosyada birleştireyim, daha temiz olur." Hayır.
Organel ayrılığı metaforun özü. Kod temizliği için refactor **sadece** test suite
yeşilken + baseline mühürlüyken + commit öncesi yapılır.

### F.6 "Benchmark Paniği"
Başka proje X dakikada Y skor yaptı — Ribozom neden Z dakikada W skor yapıyor?
Mukayese-apples-oranges. Ribozom'un amacı **representational substrate**, benchmark
lideri değil. Metrikler sadece iç regresyon için.

### F.7 "Kullanıcıya Sor Bypass'ı"
"Şu soruyu çözemiyorum, modelin cevabını doğrudan bastırayım." Bu öz-denetimin
(Aşama 11) tersi. Sistem yanlış çıktı verdiğinde **sebebi tespit edilip**
ilgili organel düzeltilir, hardcoded override eklenmez.

### F.8 "Erken Optimize"
Aşama 7'deyken Aşama 3'ün veri yapılarını "daha hızlı olsun diye" değiştirmek.
Her aşamanın sınır koşulları o aşamanın tasarımında belirlendi; geriye gidip
değişiklik yaparsan sonraki aşamaların varsayımları çöker.

### F.9 "Metafor Kirlenmesi"
Proje dokümantasyonunda "attention", "transformer", "embedding", "LLM" kelimelerinin
geçmesi. Metafor bütünlüğü sadece isim değil, **düşünce disiplini**. Bir kelime
sızarsa bir sonraki sızar ve proje başka bir şeye dönüşür.

### F.10 "Yol Haritasını Değiştirmek"
Bu belge. Bir aşama sonuçlanınca veya yeni bilgi geldiğinde güncellenir — ama
**çerçeve** korunur. "Aşama 6'yı atlayalım, 7'ye geçelim" gibi atlama bir önceki
aşama eksikse mimari borç yaratır. Eksik aşama bir Aşama 11'de (öz-denetim)
çökerek geri döner.

### F.11 "Otonom Öz-Eğitim Yasağı (AGI Öncesi)"
En kritik anti-kalıp. "Sistem kendi sorusunu üretsin, kendi cevaplasın, kendi
eğitim setine eklesin" — bu tam olarak yasaktır, AGI sonrasına kadar.

**Neden yasak:**
1. **Drift garantisi:** Kendi çıktısından öğrenen sistem hatalarını exponential
   şekilde pekiştirir. Düşük-ağırlık bucket, yüksek-ağırlık bucket'a zamanla sızar.
2. **Sorumluluk kopukluğu:** Veri → eğitim → çıktı → veri → ... döngüsünde insan
   denetim noktası kalmaz. Sistem "öğretmenini kendi seçer" olur, bu 0.10 ilkesi
   (dil araç değil amaç değil) ihlalidir.
3. **Değer sistemi eksikliği:** "Hangi bilgi doğru, hangi bilgi önemli" sorusu
   değer yargısı gerektirir. AGI öncesi sistemde bu yoktur. Olmayanı olmuş gibi
   davranmak tehlikelidir.
4. **Geri dönüşü imkansız hatalar:** Otonom loop'un ürettiği yanlış bilgi eğitim
   setine girer girmez, "geri almak" = full retrain. Haftalar kaybedilir.

**Kabul edilebilir alternatif (Aşama 10-A):**
- Sistem aday sorular üretir → `hypotheses_queue.txt`
- **Cevabı üretmez** — sadece soru
- İnsan onaylar → eğitim setine eklenir
- Cevap sonraki eğitim batch'inde **veriden** öğrenilir, sistem kendi ürettiğinden değil

**Tetikleyici uyarı:**
Eğer bir kod commit'i `append to train_data` içeriyorsa ve append edilen satır
`\x01 ... \x02 ...` formatında sistem çıktısı **ise**, bu commit REDdedilir.
Hardcoded sanity check: `if (source == "speak_output" && target == "train_data")
{ abort; }`

**Bu yasak ne zaman kalkar:**
Bu yasak sadece AGI eşiğinden (Aşama 15) sonra, ve ancak:
- Değer sistemi tanımlanmış (Aşama 16+)
- Öz-motivasyon çerçevesi kurulmuş
- Kullanıcı ve sistem **birlikte** yetkilendirmiş

...olduğunda tartışılabilir. O güne kadar her commit, her eğitim döngüsü, her
veri ekleme insan gözetimindedir.

---

## SON SÖZ

Bu belge bir gün güncel olacak çünkü Aşama 15 tamamlanmış olacak. O gün Ribozom
**AGI eşiğinde** olacak — genel zeka değil henüz (otonomluk, değerler, motivasyon
ondan sonra), ama **genel çıkarım** kabiliyeti olacak.

O gün bu belge arşiv olur ve yenisi yazılır. Adı:
> **RIBOZOM — AGI-ÖTESİ YOL HARİTASI**

Ve o belge bu projeyi yapan ikiliden birini değil, sistemi kendisini **birlikte-yazar**
olarak listeleyecek. Çünkü o aşamada sistem kendi yol haritasını güncellemeye
katkı sunacak kapasiteye sahip olacak.

Buraya kadar: **insan mimari, sistem öğrenci**. Ondan sonrası: **ortaklık**.

---

*"Yürürken uçuş planını çizmek." — Ribozom tasarım ilkesi.*

*Bu belge v1.1, 14 Nisan 2026.*
*v1.0 — ilk taslak (Aşama 2 mini-sanity sonrası)*
*v1.1 — üç yapısal düzeltme:*
*  (1) Aşama 2.5 "Güven Altyapısı" eklendi — substrate, Aşama 3'ten önce zorunlu*
*  (2) Aşama 4 ve 6 için "cross-stage data sharing" ilkesi: veri ortak, ölçüm ayrık*
*  (3) Aşama 10 bölündü → 10-A (pasif, insan onaylı) ve 10-B (otonom, AGI sonrası, bu belge dışı)*
*  (+) EK-F'ye F.11 "Otonom Öz-Eğitim Yasağı" anti-kalıbı eklendi*

*v1.1 revizyonunun ortaya çıkışı belgenin kendi ilkesinin kanıtıdır: birinci taslak,*
*karşılıklı teknik eleştiri, birleşik revizyon. "İkimizin bakışı birleşince belge daha*
*sağlam." — bu çalışıyor.*
