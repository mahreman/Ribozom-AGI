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
3. [Aşama 3 — Nedensellik](#aşama-3)
4. [Aşama 4 — Olumsuzluk ve Polarite](#aşama-4)
5. [Aşama 5 — Coreference ve Kısa-Süreli Bellek](#aşama-5)
6. [Aşama 6 — Nicelik ve Küme Mantığı](#aşama-6)
7. [Aşama 7 — Zaman](#aşama-7)
8. [Aşama 8 — Mekansal İlişki](#aşama-8)
9. [Aşama 9 — Meta-Biliş ve Güven](#aşama-9)
10. [Aşama 10 — Üretken Akıl Yürütme](#aşama-10)
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

### Aşama 2'den Aşama 3'e geçiş sinyali
- Cluster ✔ + Distribution ✔ → Aşama 3'e direkt geç
- Cluster ✔ + Distribution ❌ → Condition Layer entegrasyonu sonrası Aşama 3
- Cluster ❌ → Mimari sıçrama gerekli (bucket genişletme), sonra tekrar dene

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
## Aşama 9 — Meta-Biliş ve Güven

**Amaç:** "bilmiyorum", "eminim", "belki", "sanmıyorum" — sistemin **kendi bilgisinin
sınırını** raporlaması.

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

<a name="aşama-10"></a>
## Aşama 10 — Üretken Akıl Yürütme

**Amaç:** Sistem sadece cevap vermez, **soru üretir**, **yeni cümle kurar**, **tahmin sunar**.

### Soru
Verilmiş bir bağlamda sistem eksik bilgiyi tespit edip **kendi sorusunu** üretebilir mi?
`Ali yağmurda kaldı.` → sistem `Ali ıslandı mı?` sorusunu üretebilir mi?

### Yeni organel: **Autopoiesis Loop (Kendi-Kendini-Besleyen Döngü)**
```c
typedef struct {
    int trigger_wid;          /* bağlamı tetikleyen kelime */
    char generated_q[MAX_LINE];
    char generated_a[MAX_LINE];
    float plausibility;       /* kendi güveni */
    int verified;             /* Lizozom'da eşleşen şablon var mı? */
} synthesized_qa_t;

/* Yeni bir cümle işlendiğinde çağrılır */
void autopoiesis_probe(const char *sentence);
```

### Mekanizma
1. Cümlede **eksik slot** tespit et (`Ali yağmurda kaldı` → nesne belirsiz, durum belirsiz)
2. Mikrotübül'ü kullanarak olası **tamamlayıcı sorular** üret
3. Kendi bildiğinle cevapla (iç SPEAK)
4. Cevap plausible ise hafızaya kendi öğretmen verin olarak ekle

### Risk
Bu **yanlış-besleme** riski taşır: sistem kendi hatalarını doğrular ve sabitler.
Çözüm: autopoiesis çıkarımları **ayrı** confidence bucket'ta tutulur (düşük ağırlık),
sadece **dış onay** (insan feedback, yeni veri) ile yükselir.

### Test protokolü
1. Sistem verilen bağlamda mantıklı bir follow-up soru üretebiliyor mu?
2. Ürettiği sorulara verdiği cevaplar, eğitim verisiyle tutarlı mı?
3. Kendi-besleme loop'u divergence yaratıyor mu? (Sanity: 100 iterasyon sonra
   model hâlâ baseline batch metriklerinde mi?)

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
| Sitoiskelet | Cytoskeleton | Kausal graf | 3 | 📋 |
| Polarite Alanı | Membrane charge | Negasyon/polarite | 4 | 📋 |
| Vesicle | Vesicle | Kısa-süreli bağlam | 5 | 📋 |
| Peroksizom | Peroxisome | Küme/quantifier | 6 | 📋 |
| Kronofor | Chronophore | Zaman ekleri | 7 | 📋 |
| Spatiosome | (yeni metafor) | Uzam ilişkileri | 8 | 📋 |
| Confidence Layer | Metacognitive | Güven/belirsizlik | 9 | 📋 |
| Autopoiesis Loop | Autopoiesis | Öz-besleme | 10 | 📋 |
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

*Bu belge v1.0, 14 Nisan 2026. Son güncelleme: Aşama 2 mini-sanity sonrası.*
