# Causal Core v1.6 — Checkpoint

**Tarih:** 14 Nisan 2026 (3. gün kapanış, Aşama 7 Faz A+B mühür sonrası)
**Durum:** stabil, regression-safe, meta-biliş + kronos (tense izole).

## Kapsam (v1.5 üzerine eklenenler **kalın**)

- Aşama 3 (Kausal graf, dual CSR + BFS)
- Aşama 3.5 (Multi-hop + additive chain)
- Aşama 4 (Polarite, K3)
- Aşama 4.5 (SPEAK × Kausal köprü F1/F2/F3)
- Aşama 5 Faz A (Vesicle + why-chain)
- Aşama 5 Faz B (Chain explain — forward/backward)
- Aşama 5 Faz C (Composite organelle proof)
- Aşama 9A+9B (Meta-biliş "bilmiyorum" — üçlü-sıfır + raw confidence)
- Aşama 9C (Confidence feedback calibration — asimetrik T drift)
- **Aşama 7 Faz A (Tense detector — izole morfoloji)**
- **Aşama 7 Faz B (Vesicle.tense alanı + --tense-observe)**

## Yetenek (v1.5 satır 1–18 korunur, yeni satırlar)

| # | Özellik | Kanıt |
|---|---------|-------|
| 19 | Baseline (v72_muhur) | Test 58.2, QA 65.9% |
| 20 | **Tense detector (7A)** | 5/5 fiil: yedi=PAST, okuyor=PRESENT, yağacak=FUTURE, yaşamış=INFER, doğar=AORIST |
| 21 | **Vesicle tense kayıt (7B)** | 3/3: kedi wid=251 × {PAST, PRESENT, FUTURE} |

## "Zamanı hissetme" davranışı

```
./ribozom_v31_v7.exe --chrono-diag "kedi dün balık yedi"
  [kedi = PAST]   (morfolojik FP — Faz A sınırı)
  [dün = UNKNOWN]
  [balık = UNKNOWN]
  [yedi = PAST]   ← hedef doğru

./ribozom_v31_v7.exe --tense-observe "kedi balık yiyor"
  fiil="yiyor" tense=PRESENT
  özne="kedi" wid=251 -> vesicle_push_tense(SUBJECT, PRESENT)
  slot[0]: wid=251 role=SUBJ seq=1 tense=PRESENT
```

## Mimari kazanım

**İzolasyon.** chrono modülü tamamen bağımsız — ne rpredict, ne
SPEAK, ne kausal graf çağırır. Yalnız `chrono_detect(word, wlen)` ve
`chrono_diag_sentence(s)`. Unity-build'de chrono → vesicle → causal
sırasıyla include; vesicle_log artık chrono_name kullanarak tense
yazdırıyor.

**Suffix-reverse-match algoritması:**
1. UTF-8 → ASCII indirgeme (ç→c ğ→g ı→i ö→o ş→s ü→u)
2. FUTURE erken taraması (-ecek/-acak/-eceg/-acag son 6 karakterde)
   — peel_person'dan ÖNCE (yoksa 1pl -k, -ecek'in -k'sını yiyor)
3. peel_person: siniz/sunuz/lar/ler/sin/sun/iz/uz + tek-harf m/n/k
4. Öncelik sırası: PRESENT > FUTURE > INFER > PAST > AORIST

**Vesicle.tense alanı.** vesicle_push(wid, role) geri-uyumlu kalır
(tense=0 UNKNOWN default). Yeni vesicle_push_tense(wid, role, tense)
ile 7B, eski causal F3 push'ları bozulmaksızın koşuyor. ribo31.bin
dokunulmadı — Faz B sadece session-bound test modu.

## Sınırlar (Faz C'de çözülecek)

- **"kedi = PAST"** — morfolojik -di ismi/fiili ayırmıyor.
  Faz C'de POS-gate ile süzülecek (sadece fiil-POS'lu token tense
  alır).
- **"yağmur = AORIST"** — -r ekli isimlerde yanlış-pozitif. Aynı
  POS-gate çözer.
- Faz B subject seçimi tense filtresiz — yalnız verb konumu dışlanıyor.

## Reprodüksiyon

```sh
cd v3/files
cp checkpoint_causal_core_v16/ribo31.bin .
cp checkpoint_causal_core_v16/ribo31_causal.bin .

# Faz A — detector canlı diag
./ribozom_v31_v7.exe --chrono-diag "kedi dün balık yedi"
./ribozom_v31_v7.exe --chrono-diag "çocuk kitap okuyor"
./ribozom_v31_v7.exe --chrono-diag "yarın yağmur yağacak"
./ribozom_v31_v7.exe --chrono-diag "dinozorlar yaşamış"

# Faz B — vesicle tense kaydı
./ribozom_v31_v7.exe --skip-test --tense-observe "kedi balık yedi"
./ribozom_v31_v7.exe --skip-test --tense-observe "kedi balık yiyor"
./ribozom_v31_v7.exe --skip-test --tense-observe "kedi balık yiyecek"

# v1.5 regresyonları hâlâ geçer
sh _asama45_faz_d.sh     # 30/30
sh _trace_conf.sh        # 8-soru conf trace
sh _test_9c.sh           # T drift kalibrasyon
```

## Artefaktlar (bu dizin)

- `ribozom_v31_v7.exe` — derlenmiş binary (gcc -O2, Ubuntu WSL)
- `ribo31.bin` — baseline model (v72_muhur, dokunulmadı)
- `ribo31_causal.bin` — kausal graf
- `ribozom_chrono.{h,c}` — **yeni**: tense_t enum, chrono_detect,
  chrono_diag_sentence (~180 satır)
- `ribozom_vesicle.{h,c}` — slot.tense, vesicle_push_tense,
  vesicle_last_tense eklendi
- `ribozom_causal.{h,c}` — değişiklik yok
- `ribozom_main.c` — `--chrono-diag`, `--tense-observe` CLI flag'leri
- `ribozom_qa.h` — değişiklik yok (v1.5'ten)

## Konfig sabitleri (v1.5'ten değişiklik yok)

- CAUSAL_DEPTH_LIMIT=3, HOP_DECAY=0.7, CHAIN_MAX_HOPS=4
- EDGES_PER_NODE=32, VESICLE_SLOTS=5
- Causal short-circuit conf floor: 0.30
- K3: K1=0.45 K2=0.35 K3=0.20
- BILMIYORUM_CONF_T=0.30, T_STEP=0.02, T_CEILING=0.70
- **tense_t enum:** UNKNOWN=0, PAST=1, PRESENT=2, FUTURE=3,
  AORIST=4, INFER=5

## CLI yeni flag'ler

```
--chrono-diag "cümle"        : cümleyi tokenlara ayır, her token için
                               chrono_detect çağır, [kelime=TENSE]
                               haritasını bas. Model yüklemez, erken çıkar.
--tense-observe "cümle"      : model yükle, cümlenin ana fiilinin tense'ini
                               tespit et, özneyi vocab'dan lookup et,
                               vesicle_push_tense(SUBJECT, tense), logla.
```

## Üç gün özeti — 15 aşama/alt-faz, sıfır regresyon

| # | Aşama | Kanıt |
|---|-------|-------|
| 1 | SOV | "kartal eti yer" |
| 2 | Koşul | Temsil ✓, kullanım ✗ (belgelendi) |
| 3 | 2.5 Güven | 5-kanal motor, K1/K2 |
| 4 | 3 Kausal | Dual CSR, 30/30 |
| 5 | 3.5 Multi-hop | fırtına→soğuk→kar |
| 6 | 4 Polarite | Contrapositive, 33/33 |
| 7 | 4.5 Entegrasyon | SPEAK×Kausal, 30/30 |
| 8 | 5A Vesicle | Why-chain kar←soğuk←fırtına |
| 9 | 5B Chain | 3-hop forward domino |
| 10 | 5C Composite | 7 organel eş zamanlı |
| 11 | 9A+9B Meta-biliş | "bilmiyorum" (mars sorusu) |
| 12 | 9C Kalibrasyon | T drift, asimetrik |
| 13 | 7A Tense | 5/5 fiil tespiti |
| 14 | 7B Vesicle Tense | 3/3 tense kaydı |

Sıradaki dallanma: **Aşama 7 Faz C — tense-aware SPEAK** (köprü
çıktısında zaman uyumu; "çünkü soğuk oldu" vs "çünkü soğuk olur").
POS-gate ile 7A'nın morfolojik yanlış-pozitifleri (kedi=PAST,
yağmur=AORIST) süzülecek. Eğitim verisi dağılımı:
PRESENT 1603, AORIST 2229, PAST 1681, FUTURE 1090, INFER 0.
