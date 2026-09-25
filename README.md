PHASE 2I — FROZEN 2H REAL XAUUSD BACKTEST — RINGKAS

FROZEN: StrategyConfig() + ReversalConfig() default, sr_zones.build_zones (sorted-gap merge_atr_mult*ATR, width cap 2*zone+ (n-1)*merge), reversal.detect_at/to_trade, backtest.simulate_exit/path_ambiguity_bound. Tidak ubah threshold/indikator/optimasi. Chronological, M15/M5 open+interval<=close closed-only, no look-ahead (bars[..end_idx]), entry di close engulfing, exit walk i+1, true engulfing tetap confirmation, trending tanpa valid reversal = NO TRADE.

1. Data — REAL, bukan synthetic
data/XAUUSD_m_M5.csv (212.625) + data/XAUUSD_m_M15.csv (70.892). XAUUSD_M5_history.csv tidak dipakai (format ;, hanya overlap 2026-07/08). Setelah common_overlap+clip: M5=212.623 M15=70.892, iv M5=300s M15=900s.

2. Periode (common, naive broker time)
2023-08-28 01:00:00 .. 2026-08-26 23:45:00

3. Jumlah bar & split kronologis (bar-count, 60/20/20)
TRAIN 0..127573 (127.573 bar, 2023-08-28..2025-06-16 08:10) | VAL 127573..170098 (42.525, 2025-06-16..2026-01-21) | OOS 170098..212622 (42.524, 2026-01-21..2026-08-26). Warmup 404 HTF. OOS tidak dipakai pilih rule.

4. TRAIN | 5. VALIDATION | 6. OOS | 7-9. Setup / Win / DD
evaluated*	valid signal	BUY/SELL	NO TRADE	unresolved	wins/losses	win%	avg win/loss R	avg RR	expectancy R	PF	total R	maxDD R	cLoss/cWin
TRAIN	126.359	57	35/22	126.302	0	25/32	43.9	+1.779/-1.025	2.00	+0.205	1.36	+11.67	9.52	7/4
VAL	42.525	8	5/3	42.517	0	1/7	12.5	+1.904/-1.010	2.00	-0.645	0.27	-5.16	6.06	6/1
OOS	42.524	12	7/5	42.512	0	4/8	33.3	+1.952/-1.745	2.00	-0.513	0.56	-6.15	10.04	3/1
FULL	211.408	77	47/30	211.331	0	30/47	39.0	—	2.00	+0.005	1.01	+0.35	17.99	7/4
*evaluated=bar dengan konteks HTF valid. Signal rate TRAIN 0.045% — selektif sesuai qualified_zones+engulf 6 rules+rejection+broken. Path bound assumed=0 width=0.0000 flipped=0 semua split — exit tidak ada yang undecidable.														
Top reject TRAIN: engulf_not_at_zone 82k, zone_too_wide 57k, not_true_engulfing 43k, zone_unproven 38k, no_relevant_zone 22k, zone_broken 8.2k — selektivitas, bukan tuning.

Per pattern/SR: TRAIN SUPPORT n35 exp+0.355 PF1.67 vs RESISTANCE n22 exp-0.034; VAL terbalik SUPPORT -1.008 vs RES -0.041; OOS terbalik SUPPORT -0.158 vs RES -1.009. Per regime M15: TRAIN BULL n27 +0.424 BULL, BEAR n14 -0.012 SIDE n16 +0.025 (semua <30 INSUFF); OOS BULL n7 +0.26 BEAR n4 -1.74 SIDE n1 -1.01 — variance tinggi. **First vs repeated (touches):** 2-3 vs 4+ — TRAIN 0.097 vs 0.272 **BETTER**, VAL -1.012 vs -0.523 **BETTER**, OOS -1.007 vs +0.475 **BETTER** — repeated tidak lebih buruk, klaim sebaliknya tidak terbukti (n kecil). **Support vs Resistance** & **M15 aligned vs non-aligned flipping**: TRAIN ALIGNED n15 exp+0.675 > NON n42 +0.037; OOS ALIGNED n4 -1.74 < NON n8 +0.10 — tidak stabil.

8. Win rate & expectancy per split
Di tabel. Tidak klaim profitable/edge — hanya TRAIN >30 dan positif kecil, VAL/OOS negatif.

9. Max drawdown
Di tabel. FULL 17.99R (close-order).

10. Stabil?
TIDAK stabil. TRAIN +0.205 → VAL -0.645 → OOS -0.513 (Δ train→oos -0.718R). FULL netral +0.005 PF1.01. Performa runtuh di OOS. Sangat bergantung sedikit trade — VAL n8 OOS n12 (< MIN_SAMPLE 30 → INSUFFICIENT DATA), 1 trade ±1.9R geser exp ±0.15R. Trend filter tidak bias: trending tanpa reversal = NO TRADE via detect_at (no_rejection/not_true_engulfing 378+111+168) + require_trend_align=True; filter ADX trending_no_trade opt-in tidak dipakai pada pengukuran ini, jadi tidak ada seleksi bias.

11. Causality / regression
test_phase2h_sr_zones + test_phase2h_chart_audit + test_lookahead 55 passed. + test_reversal/test_reversal_quality 206 passed. M15 last_closed_idx open+interval <= decision_close, M5 detect_at end_idx closed, zones/ATR/rejection/engulf bars[..end_idx], touch_episodes 0..end_idx truncation-invariant/deterministic, bar_close_time bukan next-open (anti gap leak).

12. Execution safety
Domain (sr_zones/reversal/pattern_learning/filters) order_send=0 order_check=0 execution_attempts=0 (hanya header comment No execution imports di sr_zones.py). mt5_execution.py adalah modul terpisah — bukan jalur backtest. imports_execution=False. Tidak ada MT5 live order dijalankan.

13. git status — TIDAK add/commit/push
 M python/xausr/filters.py
 M python/xausr/indicators.py
?? PHASE_2F_IMPLEMENTATION_REPORT.md
?? PHASE_2G_IMPLEMENTATION_REPORT.md
?? python/tests/test_pattern_learning.py
?? python/tests/test_phase2h_chart_audit.py
?? python/tests/test_phase2h_sr_zones.py
?? python/xausr/pattern_learning.py
?? python/xausr/sr_zones.py
diff --stat 2 files 95 insertions. Log console hanya di C:\Users\ACER\AppData\Local\Temp\opencode\phase2i_console.log — tidak disimpan ke mt-info sesuai instruksi.

STOP.

