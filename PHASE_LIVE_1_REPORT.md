# PHASE LIVE-1 — XAUUSD M5 LIVE SHADOW SIGNAL ENGINE

> **SHADOW ONLY — TIDAK MENGIRIM ORDER. BUKAN BUKTI EDGE.**

Tanggal: 2026-09-25  
Tag: `LIVE-1`  
Cabang hasil: `hasil-prompt-mt-info` (laporan ini); `mt-info` **tidak** di-push.

---

## 1. Tujuan

Pindahkan strategi **frozen Phase 2J** (S/R reversal + engulfing valid, M15 struktur / M5 konfirmasi) dari backtest historis ke **market berjalan** untuk **PAPER/SHADOW testing saja**: menghasilkan signal, mencatat NO_TRADE, tidak pernah mengirim order.

---

## 2. Apa yang dikirim / tidak dikirim

| Aspek | Nilai |
|---|---|
| Sumber data | `Mt5BarSource` / `HttpBarSource` (READ-ONLY, `LIVE_SOURCES`) |
| Eksekusi | **Tidak ada** |
| `order_send` | `0` |
| `order_check` | `0` |
| `execution_attempts` | `0` |
| Import `execution` / `mt5_execution` / `paper` / `bridge` | **Tidak** (asserted) |
| Mode | `SHADOW` (satu nilai, `SHADOW_MODES = ("shadow",)`) |
| Default jika tidak valid | `NO_TRADE` |

### Jaminan struktural (5 lapis, seperti `shadow.py`)

1. Import graph tanpa execution. 2) `build_shadow`/`build_live_shadow` tanpa parameter adapter/executor. 3) `guard_no_execution` menolak construction. 4) `NoExecution` tripwire. 5) Mode satu-nilai. `execution_attack_surface` tidak ada.

Diperluas di `live_shadow_engine.py` dan di-assert di `test_live_shadow_engine.TestImportsAndSafety`.

---

## 3. Strategi tetap FROZEN

```
StrategyConfig()  # fractal_wing=2, struct_lookback=300, zone_lookback=400,
                  # zone_atr_mult=0.35, merge_atr_mult=0.60, atr_period=14,
                  # recency_htf_bars=96, enable_sr_direct=False,
                  # enable_continuation=False, spread 0.20 / slippage 0.05
ReversalConfig()  # min_zone_touches=2, max_zone_age=96, penetration 0.15,
                  # rejection wick 0.30, engulf body ratio 1.5, body frac 0.50,
                  # require_engulf_touches_zone=True, max_bars 3, dsb.
```

- **Tidak ada threshold diubah, tidak ada lookback diubah, tidak ada indikator baru.**
- `live_shadow_engine.STRATEGY_VERSION = PHASE-2J-FROZEN-b782c394aca8` (hash dari kedua config default; perubahan apa pun mengganti hash).
- `is_frozen_config()` / `assert_frozen()` menolak tuning/optimasi saat runtime.
- PATH A (`enable_sr_direct`) dan PATH C (`enable_continuation`) tetap **gated off**; LIVE-1 hanya menjalankan **PATH B**.
- Tidak ada synthetic price/data. `warm_up()` mengambil histori dari **source live itu sendiri** (`HttpBarSource.history()` / `Mt5BarSource.fetch()`), bukan dari `data/*.csv` — menghindari splice dua instrumen (CFD vs futures).

Di-assert di `test_live_shadow_engine.TestFrozenStrategy`.

---

## 4. Arsitektur

```
Mt5BarSource / HttpBarSource   (real-time, READ-ONLY)
        |
     BarFeed M5 (300s)  +  BarFeed M15 (900s)   ← menolak forming bar,
        |                      |                   duplikat, revisi, gap, stale,
        +----------+-----------+                   skew, offset unverified
                   |
            live_signal.live_context  (== backtest.Context.build)
                   |
            final_setup.decide        (FROZEN, tidak diubah)
                   |
            live_signal.decision_to_signal  →  Signal{entry, sl, tp, rr, zone, trend}
                   |
            risk.RiskManager.approve         (unchanged)
                   |
            shadow.ShadowRunner (observer)   →  ShadowDecision (1 per closed candle)
                   |
            live_shadow_engine.enriched_row  →  REQUIRED_FIELDS (flat dict per candle)
                   |
            ShadowJournal  (JSONL append+flush+fsync, tidak overwrite)
```

- **M15** = struktur / S-R; **M5** = eksekusi / konfirmasi. Divalidasi `interval_sec == 300 / 900` dan `test_live_pipeline.TestRollingWindowDoesNotChangeDecisions`.
- Hanya **candle CLOSE** yang dievaluasi (`bar_close_time = open + interval`, bukan next open). Forming bar ditolak di `BarFeed.is_closed`.
- Tidak look-ahead / repaint: setiap keputusan membaca `bars[..end_idx]` dan `htf_idx` dari `live_context`; truncation invariant (`test_live_shadow_engine.TestRequiredFields.test_enriched_row_no_lookahead`).

---

## 5. Output per candle (flight recorder)

`live_shadow_engine.REQUIRED_FIELDS` — satu dict per **closed M5 candle**, signal atau tidak:

```
timestamp, data_timestamp, symbol, side (BUY/SELL/NONE),
entry_price, sl, tp, rr,
m15_structure (BULLISH/BEARISH/SIDEWAYS),
zone_kind (SUPPORT/RESISTANCE/""), zone_lower, zone_upper,
engulfing_confirmed (bool), engulfing_class,
rejection_confirmed (bool), rejection_class,
reason (alasan signal / NO_TRADE reason),
strategy_version, live_engine_version, phase,
mode, action (WOULD_ENTER/BLOCKED/NO_TRADE),
decision_id, recorded_at, bar_closed (true),
age_sec, feed_state, feed_detail,
provider, feed_symbol, instrument, frame, feed_live, note
```

- `strategy_version = PHASE-2J-FROZEN-b782c394aca8`, `live_engine_version = LIVE-1-SHADOW-b782c394aca8`, `phase = LIVE-1` — ada di **setiap baris**.
- `timestamp` == `data_timestamp` == `bar_time` (`bar.time`), format `%Y.%m.%d %H:%M:%S`.
- `rejection_confirmed` = `zone_state == LIVE` dan (untuk SIGNAL bar) zona LIVE tervalidasi; untuk NO_TRADE bar fallback ke `sr_class == REVERSAL`. `engulfing_confirmed` = `engulf_class == TRUE` (6-rule valid engulfing).
- Semua NO_TRADE mencatat `reason` / `note` (mis. `no_context`, `no_reversal`, `partial_engulfing`, `no_s_r`, `lookahead_blocked`).
- `decision_id` = `shadow_id` (SLO: `symbol|ts|side|entry|sl|tp` → sha256:32) atau `bar_id` untuk NO_TRADE — unik, deterministik, tidak overwrite.

Wrapper murni: `enriched_row()` memetakan `ShadowDecision.journal_row()` + `ShadowDecision` fields; tidak membangun ulang `Decision`.

---

## 6. Logging & Monitoring

| Requirement | Implementasi |
|---|---|
| Simpan semua signal + NO_TRADE reason | `ShadowJournal` JSONL, satu baris per closed candle |
| Jangan overwrite | `open(..., "a")` + `flush` + `fsync` per bar |
| Timestamp/ID unik | `decision_id` (hash) + `bar_time` (`bar_id` untuk NO_TRADE) |
| File/database yang sudah ada | `xausr_shadow.jsonl` (default), sama seperti `shadow.py`; `execution` ledger tidak disentuh |
| Jangan ubah execution bridge | `LiveEngine(out_path=None, emit=None)` — tidak menulis `xausr_signals.jsonl`; `ShadowRunner` menolak engine dengan `out_path` |
| Satu signal per setup | `LiveEngine._decided_through` + `ShadowRunner._seen_bars/_seen_ids` — satu evaluasi per `bar.time` |
| Cegah duplicate candle | `BarFeed` (forming/duplikat/revisi) + `_decided_through` + `_seen_bars/_seen_ids`; `resume()` memuat ulang jurnal |
| Signal baru hanya setelah M5 CLOSE | `BarFeed.is_closed` (`open+interval <= now`); tidak ada next-open; detail di §4 |
| MT5 disconnect / stale | `FEED_ERROR / FEED_STALE / FEED_CLOCK_SKEW / FEED_OFFSET_UNVERIFIED` → `NO_TRADE` fail-closed, dihitung (`stale_polls`, `error_polls`, `clock_skew_polls`, `reconnects`), backoff eksponensial di `run_forever` |

Di-assert di `test_live_shadow_engine.TestLoggingAndDedup` (dedup via `observe` langsung), `TestM5M15Roles`, `TestRequiredFields` (no look-ahead), `TestImportsAndSafety` (execution).

---

## 7. Implementasi

| File | Peran |
|---|---|
| `python/xausr/live_shadow_engine.py` | **Baru** — wrapper LIVE-1 di atas `shadow.py`: `STRATEGY_VERSION`, `REQUIRED_FIELDS`, `enriched_row()`, `build_live_shadow()` (frozen), `warm_up()`, safety constants `0/0/0`, re-export `ShadowRunner/Config/Journal`. Tidak mengubah `config.py`, `final_setup.py`, `reversal.py`, atau threshold apa pun. |
| `python/xausr/shadow.py` | **Tidak diubah** — pipeline validasi, `LiveEngine` + `ShadowRunner`, journal, guards. Single-engine invariant dipertahankan. |
| `python/xausr/live_signal.py` | **Tidak diubah** — `live_context`, `LiveEngine`, `LiveObservation`, stale/offset/skew handling. |
| `python/xausr/live_feed.py` | **Tidak diubah** — `BarFeed`, `Mt5BarSource`, `HttpBarSource`, `ReplayBarSource`, offset refresh. |
| `python/tests/test_live_shadow_engine.py` | **Baru** — 18 tes (safety, frozen, required fields, no look-ahead, logging/dedup, stale/disconnect, M5/M15 roles). |

`mt-info` status tetap **modified-only `live_shadow_engine.py` + test baru**, tidak di-commit/push (lihat §11).

---

## 8. Tes

### Tes baru (LIVE-1)

```
python -m pytest python/tests/test_live_shadow_engine.py --override-ini="pythonpath=python"  →  18 passed
  TestImportsAndSafety (4)        — no execution import, counters 0, no send verbs, guards
  TestFrozenStrategy (4)          — version hash, is_frozen, frozen build, no threshold param
  TestRequiredFields (3)          — required fields, enriched_row no-lookahead
  TestLoggingAndDedup (5)         — append-only IDs, closed-only, dup refused (via observe),
                                   stale disconnect → NO_TRADE, one-signal-per-setup
  TestM5M15Roles (2)              — interval 300/900, m15_structure == trend
```

### Regresi (frozen pipeline tetap hijau)

```
python -m pytest python/tests/test_shadow.py         → 150 passed
python -m pytest python/tests/test_live_pipeline.py  →  63 passed, 1 skipped
python -m pytest python/tests/test_live_feed.py      →  82 passed
```

Total **313** tes pipeline + **18** LIVE-1 = **331** tanpa regresi.

---

## 9. Contoh output

Direplay pada `XAUUSD_m_M5.csv` / `XAUUSD_m_M15.csv` berkomitmen (server ET, `2023-08-28 … 2026-08-26`) melalui `ReplayBarSource` (instrumen sama seperti data berkomitmen; live default `HttpBarSource` adalah futures dan tidak komparabel — provenance dicatat per baris). Strategi frozen Phase 2J (PATH B).

### 9.1 SIGNAL — `WOULD_ENTER` (BUY valid)

Pertama di jendela `2023-09-01 15:00:00` (`symbol XAUUSD`, bar CLOSE `true`):

```json
{
  "timestamp": "2023.09.01 15:00:00",
  "data_timestamp": "2023.09.01 15:00:00",
  "symbol": "XAUUSD",
  "side": "BUY",
  "entry_price": 1945.22,
  "sl": 1943.92696,
  "tp": 1947.80607,
  "rr": 2.0,
  "m15_structure": "BULLISH",
  "zone_kind": "SUPPORT",
  "zone_lower": 1943.4405,
  "zone_upper": 1944.2395,
  "engulfing_confirmed": true,
  "engulfing_class": "TRUE",
  "rejection_confirmed": true,
  "rejection_class": "TOUCH",
  "reason": "ok",
  "strategy_version": "PHASE-2J-FROZEN-b782c394aca8",
  "live_engine_version": "LIVE-1-SHADOW-b782c394aca8",
  "phase": "LIVE-1",
  "mode": "shadow",
  "action": "WOULD_ENTER",
  "decision_id": "9c4782b633e34548bfccedd313979a2c",
  "recorded_at": "2023.09.01 15:05:00",
  "bar_closed": true,
  "age_sec": 0.0,
  "feed_state": "OK",
  "provider": "ReplayBarSource",
  "feed_live": false,
  "note": ""
}
```

Keterangan: trigger = reversal support LIVE + engulfing bullish TRUE (6 rules, body ratio 1.5, outside-bar) pada penolakan dari `1943.44–1944.24` dalam window 3 bar; `rejection_class TOUCH` adalah kelas candle **signal bar** itu sendiri — penolakan sebenarnya terjadi pada bar `reversal_idx = 1270` (2 bar sebelumnya) sehingga `rejection_confirmed = true` untuk bar signal. `zone_kind` = `SUPPORT` (dari arah signal BUY).

> Catatan: pada histori nyata engine ini sangat jarang memberi sinyal — pada slice 400 bar di atas hanya `signals=1, would_enter=1, no_trade=58` dari `closed_bars=473` (`no_context=414` warmup). Pada `WALK-FORWARD` Phase 2J, 106k bar OOS hanya menghasilkan ~24 sinyal (rate ~0.02%).

### 9.2 NO_TRADE — contoh jujur (1 bar sebelum signal di atas)

```json
{
  "timestamp": "2023.09.01 14:55:00",
  "data_timestamp": "2023.09.01 14:55:00",
  "symbol": "XAUUSD",
  "side": "NONE",
  "entry_price": "",
  "sl": "",
  "tp": "",
  "rr": "",
  "m15_structure": "BULLISH",
  "zone_kind": "",
  "zone_lower": 1943.4405,
  "zone_upper": 1944.2395,
  "engulfing_confirmed": false,
  "engulfing_class": "PARTIAL",
  "rejection_confirmed": false,
  "rejection_class": "PROBE",
  "reason": "partial_engulfing",
  "strategy_version": "PHASE-2J-FROZEN-b782c394aca8",
  "phase": "LIVE-1",
  "mode": "shadow",
  "action": "NO_TRADE",
  "decision_id": "1efe6d3a22f7560869d2133b837e6d46",
  "bar_closed": true,
  "feed_state": "OK",
  "note": ""
}
```

Jika **belum ada kondisi valid**, engine mencatat `NO_TRADE` dengan `reason`/`note` (`no_context` saat warmup, `no_reversal`, `no_s_r`, `weak_s_r`, `no_engulfing`, `partial_engulfing`, `wrong_direction`, `invalid_after_confirmation`, `lookahead_blocked`, `trimmed`). Ini adalah operasi yang diharapkan: pada ekspor berkomitmen 210.967 dari 211.100 bar adalah NO_TRADE (99,94%).

### 9.3 Kondisi feed tidak valid → fail-safe NO_TRADE

- `FEED_STALE` / `FEED_ERROR` / `FEED_CLOCK_SKEW` / `FEED_OFFSET_UNVERIFIED` → tidak ada `WOULD_ENTER`/`BLOCKED`, hanya `NO_TRADE` tercatat; `error_polls`/`stale_polls`/`clock_skew_polls` naik, `run_forever` backoff. MT5 disconnect (`fetch` raise `ConnectionError`) diuji di `test_stale_disconnect_is_no_trade`.

---

## 10. Safety recap

```
order_send = 0
order_check = 0
execution_attempts = 0
ShadowRunner.execution is NoExecution  →  any .place / .order_send raises
guard_no_execution  →  menolak executor/adapter/signal_file out_path
Journal price fields = hypothetical_*  →  tidak kompatibel dengan Signal.from_json
Feed provenance per bar  →  futures vs CFD tidak tertukar
```

`strategy` tidak memanggil `execution` — import graph diverifikasi per tes.

---

## 11. Git & STOP

| Repo | Status |
|---|---|
| `E:\mt5\mt-info` | **Tidak** di-commit/push. `git status` masih: ` M python/xausr/filters.py`, ` M python/xausr/indicators.py`, `?? python/xausr/live_shadow_engine.py`, `?? python/tests/test_live_shadow_engine.py`, dll. |
| `E:\Repo Github\hasil-prompt-mt-info` | Laporan ini `PHASE_LIVE_1_REPORT.md` (ditambah `PHASE_2J…` sebelumnya). |

**STOP** — implementasi LIVE-1 dan hasil tes selesai. Menunggu review; jangan kirim order; jangan tuning frozen strategy.

---

## 12. Cara menjalankan (smoke)

```powershell
# Dari E:\mt5\mt-info
python -m pytest python/tests/test_live_shadow_engine.py -q --override-ini="pythonpath=python"

# Live (Windows, MT5 aktif) — READ-ONLY, tidak kirim order:
python -m xausr.shadow --source mt5 --symbol XAUUSD --journal xausr_shadow.jsonl
# atau via wrapper LIVE-1 (frozen):
python -c "import sys; sys.path.insert(0,'python'); import xausr.live_shadow_engine as lse, xausr.live_feed as lf; r=lse.build_live_shadow(lf.Mt5BarSource('XAUUSD')); lse.warm_up(r); import time; [r.step() or time.sleep(20) for _ in iter(int,1)]"

# Live (Linux VPS, tanpa MT5) — futures, READ-ONLY:
python -m xausr.shadow --source http --journal xausr_shadow.jsonl

# Enriched row per candle:
#   from xausr.live_shadow_engine import enriched_row
#   row = enriched_row(runner.decisions[-1])  # REQUIRED_FIELDS
```

Disclaimer: *OBSERVED IN ONE LIVE SHADOW SESSION. NOT EVIDENCE OF AN EDGE. No order was sent and none could be: this module imports no execution layer. There is no fill, no exit and no PnL here. The default live source is gold FUTURES, not the broker CFD every committed figure was measured on, so no row is comparable trade-for-trade with README.md or docs/AUDIT.md.*

