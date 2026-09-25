# PHASE 2J — EXPANDED REAL OOS + WALK-FORWARD VALIDATION

**Tanggal:** 2026-09-25
**Status:** `EDGE NOT PROVEN`
**Versi strategi:** FROZEN Phase 2H — tidak ada perubahan threshold/indikator/parameter

---

## 0. Ringkasan Eksekutif

Evaluasi frozen-rule S/R reversal (`reversal.detect_at` + `to_trade` + `sr_zones.build_zones` + `backtest.simulate_exit`) pada **seluruh histori REAL XAUUSD M5+M15** dengan expanded chronological OOS / walk-forward. Hasil **tidak menunjukkan edge yang stabil**: IS positif kecil, semua OOS window kecil dan berbalik arah, kumulatif OOS negatif, dan target `>=100` trade kumulatif OOS **tidak tercapai tanpa pelonggaran rule** — dilaporkan apa adanya.

> **EDGE NOT PROVEN** — Data real 2023-2026 tidak cukup membuktikan edge reversal ini. Strategi tetap frozen; hasil buruk tidak dipakai untuk tuning.

---

## 1. Frozen Version (tidak diubah dari Phase 2I)

- `StrategyConfig()` default: `fractal_wing=2, struct_lookback=300, zone_lookback=400, zone_atr_mult=0.35, merge_atr_mult=0.60, atr_period=14, sl_atr_mult=1.0, min_rr=1.5, min_score=60, require_trend_align=True, recency 96 HTF, spread 0.20/slip 0.05`
- `ReversalConfig()` default: `min_zone_touches=2, max_age 96 HTF, penetration 0.15, rejection wick 0.30, engulf body_ratio 1.5 / body_frac 0.50 / engulfed_body 0.10 ATR / range_cover 1.0, require_engulf_touches_zone True, max_bars_rejection_to_engulf 3, break_buffer 0.10 ATR, side_lookback 48, min_reactions 2, evidence 288, displacement 0.30 ATR`
- `sr_zones.build_zones`: sorted-gap clustering `gap <= merge_atr_mult*ATR`, mid = median, `upper = max+half, lower = min-half`, cap `max_w = (2*zone_atr_mult + (n-1)*merge_atr_mult)*ATR` simetris; flip via `_decisive_flip` order-aware causal.
- `reversal.detect_at` / `to_trade` / `simulate_exit` (stop-first, entry di close engulfing, exit walk `i+1`, cost real).
- **Tidak ada** indikator baru, tidak ada optimasi, tidak ada perubahan `execution/bridge/risk`.

---

## 2. Data REAL (WAJIB, bukan synthetic)

- `data/XAUUSD_m_M5.csv` 212.625 bar, `data/XAUUSD_m_M15.csv` 70.892 bar (repo `mt-info`).
- `data/XAUUSD_M5_history.csv` **tidak dipakai** (format `;`, hanya 2026-07/08, 10k bar, tidak konsisten dengan M15).
- Setelah `common_overlap()` + `clip_to_period()`: **M5=212.623, M15=70.892**.
- Interval via `infer_interval_sec` modal: **M5=300s, M15=900s**, `bar_close_time = open + interval` (bukan next-open, anti gap-weekend).
- Periode common (naive broker time): **2023-08-28 01:00:00 .. 2026-08-26 23:45:00**.
- H1 untuk fingerprint learning: `data/XAUUSD_m_H1.csv` clipped ke common → 17.739 bar (hanya untuk observasi, tidak memengaruhi rule).
- **Chronological only**, no shuffle, no look-ahead, forming bar excluded, `H1`/`M15` closed-only via `_htf_last_closed_index(open+interval <= decision_close)`.

---

## 3. Validasi — Expanded Chronological OOS / Walk-Forward

Dua pandangan dilaporkan (keduanya frozen, chronological, bar-count based):

**A. Walk-forward utama Phase 2J (IS anchor 50% + 4 OOS folds):**

| Window | lo | hi | waktu | bar | evaluasi |
|---|---|---|---|---|---|
| IS_50 | 0 | 106.311 | 2023-08-28 .. 2025-02-26 04:20 | 106.311 | anchor (bukan untuk tuning) |
| OOS_WF1 | 106.311 | 132.889 | 2025-02-26 .. 2025-07-11 20:45 | 26.578 | OOS |
| OOS_WF2 | 132.889 | 159.467 | 2025-07-11 .. 2025-11-25 07:55 | 26.578 | OOS |
| OOS_WF3 | 159.467 | 186.045 | 2025-11-25 .. 2026-04-14 06:20 | 26.578 | OOS |
| OOS_WF4 | 186.045 | 212.622 | 2026-04-14 .. 2026-08-26 23:45 | 26.577 | OOS |
| **Kumulatif OOS WF** | 106.311 | 212.622 | 2025-02-26 .. 2026-08-26 | 106.311 | — |

**B. Legacy 60/20/20 (komparabilitas Phase 2I):**

| Window | bar | waktu |
|---|---|---|
| TRAIN_60 | 127.573 | 2023-08-28 .. 2025-06-16 08:15 |
| VAL_20 | 42.525 | 2025-06-16 .. 2026-01-21 08:40 |
| OOS_20 | 42.524 | 2026-01-21 .. 2026-08-26 23:45 |

Target **`>=100` valid trades kumulatif OOS tidak dilonggarkan** — jika tidak tercapai, laporkan apa adanya.

---

## 4. A. Hasil Per Window (frozen-rule strategy)

Resolved trades only (`WIN/LOSS`, `OPEN` excluded, cost net).

| Window | evaluated* | valid signal | BUY | SELL | NO TRADE | unresolved | wins | losses | win% | avg win R | avg loss R | avg RR | expectancy R | PF | total R | maxDD R | cWin | cLoss | path width |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **IS_50** | 105.097 | 53 | 33 | 20 | 105.044 | 0 | 22 | 31 | 41,5 | +1,79 | -1,03 | 2,00 | **+0,130** | 1,22 | +6,89 | 9,52 | 4 | 7 | 0,0000 |
| **OOS_WF1** | 26.578 | 5 | 3 | 2 | 26.573 | 0 | 3 | 2 | 60,0 | +1,93 | -1,01 | 2,00 | **+0,755** | 2,87 | +3,77 | 2,02 | 3 | 2 | 0,0000 |
| **OOS_WF2** | 26.578 | 6 | 3 | 3 | 26.572 | 0 | 1 | 5 | 16,7 | +1,90 | -1,01 | 2,00 | **-0,525** | 0,38 | -3,15 | 5,05 | 1 | 5 | 0,0000 |
| **OOS_WF3** | 26.578 | 5 | 3 | 2 | 26.573 | 0 | 2 | 3 | 40,0 | +1,96 | -1,01 | 2,00 | **+0,178** | 1,29 | +0,89 | 2,01 | 1 | 2 | 0,0000 |
| **OOS_WF4** | 26.577 | 8 | 5 | 3 | 26.569 | 0 | 2 | 6 | 25,0 | +1,95 | -1,74 | 2,00 | **-1,006** | 0,33 | -8,05 | 9,99 | 1 | 3 | 0,0000 |
| **Kumulatif OOS WF** | 106.311 | **24** | 14 | 10 | 106.287 | 0 | 8 | 16 | 33,3 | +1,93 | -1,31 | 2,00 | **-0,272** | 0,70 | **-6,54** | 14,26 | 3 | 6 | 0,0000 |
| TRAIN_60 | 126.359 | 57 | 35 | 22 | 126.302 | 0 | 25 | 32 | 43,9 | +1,78 | -1,03 | 2,00 | +0,205 | 1,36 | +11,67 | 9,52 | 4 | 7 | 0,0000 |
| VAL_20 | 42.525 | 8 | 5 | 3 | 42.517 | 0 | 1 | 7 | 12,5 | +1,90 | -1,01 | 2,00 | -0,645 | 0,27 | -5,16 | 6,06 | 1 | 6 | 0,0000 |
| OOS_20 | 42.524 | 12 | 7 | 5 | 42.512 | 0 | 4 | 8 | 33,3 | +1,95 | -1,75 | 2,00 | -0,513 | 0,56 | -6,15 | 10,04 | 1 | 3 | 0,0000 |
| FULL | 211.408 | 77 | 47 | 30 | 211.331 | 0 | 30 | 47 | 39,0 | — | — | 2,00 | +0,005 | 1,01 | +0,35 | 17,99 | 4 | 7 | 0,0000 |

*`evaluated` = bar dengan konteks HTF valid (warmup 404 HTF, zones non-empty, ATR>0). Signal rate IS_50 0,050% — selektif sesuai `qualified_zones + 6 engulf rules + rejection + broken`.

**Jawaban target >=100:** **TIDAK TERCAPAI** — kumulatif OOS WF hanya **24** trade. Rule **tidak dilonggarkan** untuk mengejar jumlah. Dengan selectivitas ~0,05% (~1 trade per 2.000 bar evaluasi), 100 trade butuh ~200k bar OOS (~3× histori saat ini). Keterbatasan dilaporkan, bukan ditutupi.

Top reject IS_50 (mekanisme selektivitas, bukan tuning): `engulf_not_at_zone` dominan, diikuti `zone_too_wide`, `not_true_engulfing`, `zone_unproven`, `no_relevant_zone`, `zone_broken` — sama pola dengan Phase 2I.

---

## 5. B. Analisis Pola (frozen, causal)

### SUPPORT vs RESISTANCE
| Window | SUPPORT n / exp | RESISTANCE n / exp |
|---|---|---|
| IS_50 | 33 +0,259 | 20 -0,083 |
| OOS_WF1 | 3 +0,949 | 2 +0,463 |
| OOS_WF2 | 3 -1,008 | 3 -0,041 |
| OOS_WF3 | 3 -0,020 | 2 +0,474 |
| OOS_WF4 | 5 -0,411 | 3 -1,998 |

Tidak ada sisi yang konsisten unggul — flipping antar window.

### First / Repeated Touch (bucket `touches` cluster)
| Window | 2–3 n/exp | 4+ n/exp |
|---|---|---|
| IS_50 | 20 +0,061 | 33 +0,172 |
| OOS_WF1 | 2 +0,463 | 3 +0,949 |
| OOS_WF2 | 2 -1,012 | 4 -0,281 |
| OOS_WF3 | 3 -0,018 | 2 +0,471 |
| OOS_WF4 | 5 -1,600 | 3 -0,016 |

**Repeated (4+) tidak lebih buruk** — justru sedikit lebih baik di IS_50, WF1, WF2, WF3; hanya WF4 keduanya negatif. Klaim `repeated lebih buruk` tidak terbukti.

### M15 Aligned vs Non-aligned (BUY↔BULL, SELL↔BEAR)
| Window | ALIGNED n/exp | NON-ALIGNED n/exp |
|---|---|---|
| IS_50 | 14 +0,586 | 39 -0,034 |
| OOS_WF1 | 1 +1,930 | 4 +0,461 |
| OOS_WF2 | 2 -1,008 | 4 -0,283 |
| OOS_WF3 | 1 +1,957 | 4 -0,267 |
| OOS_WF4 | 3 -2,976 | 5 +0,176 |

Aligned unggul di IS_50/WF1/WF3, **runtuh WF2/WF4** (WF4 aligned -2,976R). Tidak stabil; membuktikan `non-aligned = NO TRADE` tidak memperbaiki generalisasi.

### BULL / BEAR / SIDE (M15 `classify_trend`)
| Window | BULL n/exp | BEAR n/exp | SIDE n/exp |
|---|---|---|---|
| IS_50 | 25 +0,303 | 13 -0,161 | 15 +0,094 |
| OOS_WF1 | 2 +1,933 | 2 +0,458 | 1 -1,011 |
| OOS_WF2 | 1 -1,007 | 1 -1,009 | 4 -0,283 |
| OOS_WF3 | 3 +0,968 | 1 -1,007 | 1 -1,009 |
| OOS_WF4 | 4 -0,269 | 4 -1,743 | 0 — |

BEAR konsisten lemah; BULL/SIDE flip. Semua bucket per-regime `<30` → INSUFFICIENT secara individual.

### Rejection + True Engulfing
Semua 77 trade FULL melewati `rejection_failures` (penetration ≥0,15w, close kembali di luar zone, wick ≥0,30) **dan** `is_bull/bearish_engulfing` 6 rules (containment, engulfed_body ≥0,10 ATR, ratio ≥1,5, body_frac ≥0,50, range_cover ≥1,0, extreme taken) serta `require_engulf_touches_zone`. Tidak ada trade `mid-range` — window dengan trending tanpa rejection/engulf valid tetap **NO TRADE** (378+111+168 `no_rejection` across splits di Phase 2I; sama di 2J).

### Distribusi R & Kontribusi Trade Terbesar (kumulatif OOS WF, n=24)
`min -6,92  median -1,01  max +1,96  mean -0,272  stdev 1,95`
Top-3 WIN = **+5,87R** / total abs **6,54R** = **89,8%** — hasil kumulatif **ditopang 3 trade** (fragil). 5 WIN terbesar semuanya `R_SUPPORT/RESISTANCE_REVERSAL_ENGULFING` +1,93..+1,96R. Satu loss WF4 -6,92R (gap/slip) mendominasi.

---

## 6. C. Stability (chronological)

```
IS_50  exp +0,130  (n53, satu-satunya n>30, positif kecil)
WF1    exp +0,755  (n5, 60% win — noise, n kecil)
WF2    exp -0,525  (n6, runtuh)
WF3    exp +0,178  (n5, rebound)
WF4    exp -1,006  (n8, runtuh terdalam)
Kumulatif OOS exp -0,272  PF 0,70
Legacy: TRAIN +0,205 → VAL -0,645 → OOS -0,513 (Δ train→oos -0,718R)
```

**Chronological tidak konsisten.** WF1 positif didorong 3 WIN beruntun, langsung dibatal WF2, WF4 menghapus semua gain. Tidak ada drift monotonic yang menjelaskan — murni variance pada n kecil. **Semua OOS window INSUFFICIENT DATA** (`n < 30` per `stats.MIN_SAMPLE`). Satu-satunya window yang bisa dinilai (`IS_50 n53`) positif kecil namun CI-nya mencakup nol (lihat §7). **Tidak boleh menyimpulkan edge** — setiap OOS window ditandai `INSUFFICIENT` dan kumulatif OOS tetap kecil (24).

**Trend filter:** `trending tanpa valid reversal = NO TRADE` bekerja via `detect_at` (rejection/engulf gagal) + `require_trend_align=True` (counter-trend diblok). Filter ADX `trending_no_trade` opt-in `FilterSet` **tidak dipakai** pada pengukuran frozen ini — jadi tidak ada pemilihan data bias dari filter tambahan. Tidak ada pengurangan false reversal yang stabil terbukti, karena tidak ada window OOS yang cukup untuk mengukurnya tanpa bias seleksi.

---

## 7. D. Statistik (tidak dipakai untuk mengubah strategi)

**Wilson 95% CI win rate, bootstrap 95% CI expectancy (10k resample, seed 42), hanya bila `n≥10`:**

| Window | n | win% | Wilson 95% win | exp R | bootstrap 95% exp | PF | DD R |
|---|---|---|---|---|---|---|---|
| IS_50 | 53 | 41,5 | **29,3–54,9** | +0,130 | **[-0,236, +0,507]** | 1,22 | 9,52 |
| OOS_WF1 | 5 | 60,0 | 23,1–88,2 | +0,755 | INSUFF (<10) | 2,87 | 2,02 |
| OOS_WF2 | 6 | 16,7 | 3,0–56,4 | -0,525 | INSUFF | 0,38 | 5,05 |
| OOS_WF3 | 5 | 40,0 | 11,8–76,9 | +0,178 | INSUFF | 1,29 | 2,01 |
| OOS_WF4 | 8 | 25,0 | 7,1–59,1 | -1,006 | INSUFF | 0,33 | 9,99 |
| TRAIN_60 | 57 | 43,9 | 31,8–56,7 | +0,205 | [-0,151, +0,568] | 1,36 | 9,52 |
| VAL_20 | 8 | 12,5 | 2,2–47,1 | -0,645 | INSUFF | 0,27 | 6,06 |
| OOS_20 | 12 | 33,3 | 13,8–60,9 | -0,513 | [-1,992, +0,718] | 0,56 | 10,04 |
| Kum OOS WF | 24 | 33,3 | 17,5–54,0 | -0,272 | [-0,92, +0,38] est | 0,70 | 14,26 |

Interpretasi: **semua CI mencakup 0 expectancy dan mencakup win% 50%** kecuali WF1 yang CI-nya 23–88% (n5). Bahkan IS_50 yang paling baik, CI exp `[-0,236, +0,507]` mencakup negatif — **tidak signifikan**. Statistik tidak dipakai untuk tuning; hanya untuk menunjukkan ketidakpastian.

---

## 8. E. Causality (regression)

- **M15 closed-only:** `bar_close_time(open+interval)` + `_htf_last_closed_index` binary search `open+interval <= decision_close` — forming M15 bar excluded, gap tidak bocor (bukan next-open).
- **M5 closed-only:** `detect_at(end_idx)` hanya `bars[..end_idx]`, entry di close engulfing, exit walk `i+1`.
- **No look-ahead/repaint:** `build_zones`, `atr_at`, `rejection_failures`, `is_bull/bearish_engulfing`, `zone_side_intact` semua `bars[..end_idx]`; `touch_episodes 0..end_idx` truncation-invariant & deterministic.
- **Path bound:** `assumed=0 width=0.0000 flipped=0` semua window — **tidak ada bar exit undecidable** (observasi `gapped` via open, bukan asumsi `stop-first`).
- **Tests:** `test_phase2h_sr_zones` + `test_phase2h_chart_audit` + `test_lookahead` **55 passed**, `+ test_reversal/test_reversal_quality` **206 passed** sebelum run ini (2026-09-25).

---

## 9. F. Safety

- **Domain backtest/strategy (`sr_zones/reversal/backtest/pattern_learning/filters/reversal_quality/reversal_rules`):** `order_send=0, order_check=0, execution_attempts=0, imports_execution=False`.
- `mt5_execution.py` / `live_signal` / `bridge` / `risk` **tidak diimport** oleh jalur backtest — modul terpisah.
- **Tidak ada live order** dikirim; `ExecutionConfig.mode=dry-run` default.
- Report ini **tidak disimpan** di `mt-info`; hanya di `hasil-prompt-mt-info` (sesuai instruksi). `git status mt-info` tetap **tidak di-add/commit/push**: `M filters.py, M indicators.py, ?? sr_zones.py, ?? pattern_learning.py, ?? test_phase2h_*`.

---

## 10. Observation / Learning (terpisah dari frozen rule)

Fingerprint `pattern_learning.extract_fingerprint_v2(use_sr_zones=True)` kausal, `label_outcome` terpisah (tidak masuk feature), `temporal_splits` chronological:

- Observasi diambil **hanya pada 24 trade kumulatif OOS** (entry-time fingerprint, H1+M15 closed-only).
- `learn_patterns(train 60 / val 20 / oos 20)`: **0 patterns dengan `n_total≥5`**, `VALIDATED=0 CANDIDATE=0` — sample terlalu kecil untuk pembelajaran yang berarti.
- **Tidak mengubah strategy rule** — learning murni observasi.

---

## 11. Verdict

### EDGE NOT PROVEN

Alasan berbasis data:

1. **Kumulatif OOS WF n=24 exp -0,272R PF 0,70** — negatif, `n` jauh di bawah 100, tidak signifikan.
2. **Semua OOS window INSUFFICIENT** (n 5–8, satu 12) — CI Wilson 23–88% / bootstrap mencakup nol — hasil didominasi noise.
3. **Chronological tidak stabil** — WF1 +0,75 → WF2 -0,52 → WF3 +0,18 → WF4 -1,01 — flipping tanpa persistensi; legacy TRAIN +0,205 → VAL -0,645 → OOS -0,513 runtuh.
4. **Top-3 trade menopang 90% total** — fragil, bukan edge yang terdistribusi.
5. **Bahkan IS_50 terbaik (n53 exp+0,13) CI mencakup nol** — tidak bisa diklaim profitable.
6. **Tidak ada pola S/R/regime/alignment yang konsisten** — SUPPORT/RESISTANCE, BULL/BEAR, aligned/non-aligned semua flip antar window.
7. **Target 100 OOS tidak tercapai tanpa pelonggaran** — dilaporkan jujur; pelonggaran akan menjadi optimasi yang dilarang.

Tidak ada perubahan strategi dilakukan karena OOS buruk — freeze dihormati. Langkah lanjut yang valid (bukan tuning) adalah **kumpul lebih banyak histori real** atau **hentikan klaim edge** sampai n OOS ≥100 membuktikan sebaliknya.

---

## 12. Lampiran Teknis

- `common_overlap` M5/M15, `clip_to_period`, `infer_interval_sec` modal.
- Cost: `spread 0.20 (half 0.10) + slip 0.05 + comm 0.00` per sisi.
- R per trade = `(exit - fill_entry - comm)/risk`, `risk = |fill_entry - SL|`.
- Metrics `st.metrics` close-order, `max_drawdown_R`, `stdev_R`, `max_consec`, `path_ambiguity_bound` stop-first vs target-first.
- Random seed bootstrap 42, Wilson z=1,96.

---

## 13. Git

- `mt-info`: **TIDAK** `add/commit/push` (instruksi). Status: `M filters.py, M indicators.py, ?? sr_zones.py, ?? pattern_learning.py, ?? PHASE_2F/2G md, ?? test_phase2h_*`.
- `hasil-prompt-mt-info`: report ini (`PHASE_2J_IMPLEMENTATION_REPORT.md`) + `README.md` Phase 2I. Push dilakukan ke `https://github.com/zenolambee/hasil-prompt-mt-info` saja.

---

*End of report — frozen strategy, real data, chronological, no look-ahead, no optimization, no execution.*
