# PHASE LIVE-2 — Windows MT5 XAUUSD Shadow Run

> **SHADOW ONLY — tidak ada order dikirim. Hasil ini hanya smoke run operasional, bukan bukti edge.**

Run: 2026-09-25, terminal server time UTC+3  
Engine version: `LIVE-1-SHADOW-b782c394aca8`  
Strategy version: `PHASE-2J-FROZEN-b782c394aca8`

## 1. Koneksi dan instrumen

- **MT5 connected:** YES (`terminal_info.connected=True`, MT5 build 6182).
- **Broker/server:** JustMarkets-Demo2.
- **Broker symbol yang dipakai:** `XAUUSD.m` (visible; tick bid/ask tersedia). Simbol persis `XAUUSD` tidak tersedia pada terminal ini.
- **Feed:** `Mt5BarSource`, read-only. Tick clock mengonfirmasi server offset **UTC+3**. Mode `shadow`; engine tidak membuat signal output file/bridge atau execution adapter.
- **Trade permission:** tidak dipakai oleh engine. Run hanya memanggil initialize, symbol/tick/history reads dan `copy_rates_from_pos`; tidak memanggil order API.

## 2. Live shadow run (run dengan warm-up)

Engine menggunakan histori dari MT5 terminal yang sama untuk warm-up; tidak memakai CSV atau synthetic data.

| Metrik | Hasil |
|---|---:|
| Warm-up M5 closed bars | 1,999 |
| Warm-up M15 closed bars | 999 |
| Interval | M5 300s / M15 900s |
| Polls | 12 selama ~3m29s |
| M5 closed bars baru diterima & dinilai | **1** |
| SIGNAL | **0** |
| NO_TRADE | **1** |
| Duplicate candle | 0 |
| M5 forming bars ditolak | 13 |
| stale / error / clock-skew / offset-unverified | 0 / 0 / 0 / 0 |
| Observer / journal errors | 0 / 0 |

Polling menunggu hingga ada M5 candle baru yang benar-benar closed. Bar close `2026-09-25 20:05:00` (broker server time) diterima pada 20:10:10, umur ~10.2 detik.

### Live NO_TRADE yang tercatat

```json
{
  "timestamp": "2026.09.25 20:05:00",
  "symbol": "XAUUSD.m",
  "side": "NONE",
  "m15_structure": "BEARISH",
  "zone_lower": 4284.19375,
  "zone_upper": 4304.36625,
  "engulfing_class": "NONE",
  "rejection_class": "PROBE",
  "reason": "no_reversal",
  "strategy_version": "PHASE-2J-FROZEN-b782c394aca8",
  "data_timestamp": "2026.09.25 20:05:00",
  "bar_closed": true,
  "feed_state": "OK",
  "provider": "Mt5BarSource",
  "feed_symbol": "XAUUSD.m",
  "feed_live": true,
  "sent": false,
  "position_opened": false
}
```

Tidak ada signal valid pada candle tersebut; engine tidak memaksakan signal.

## 3. Logging dan satu-keputusan-per-candle

- **Record live run:** 1 bar, timestamp unik, closed, provider dan broker symbol tercatat, `sent=false`, `position_opened=false`.
- Feed mengabaikan 13 forming bars; 1 candle baru menghasilkan tepat 1 keputusan. Feed `IDLE` pada poll lainnya dan tidak menulis duplikat.
- **Catatan integritas journal:** setelah live run selesai, verifikasi read-only tambahan sempat membangun runner baru tanpa warm-up untuk mengecek ulang feed. Runner diagnostik itu menambahkan 48 bar catch-up lama bertanda `note=no_context` ke file yang sama; seluruhnya `age_sec > 300`, bukan bar live yang dinilai strategi, tidak menghasilkan signal dan tidak dihitung di tabel live-run di atas. Baris-baris itu dipertahankan (tidak dihapus/ditimpa) demi append-only dan harus diperlakukan sebagai diagnostik stale, bukan hasil run. Baris run live adalah record close `20:05:00` dengan reason `no_reversal`.


```text
order_send       = 0
order_check      = 0
execution_attempts = 0
orders_sent      = 0
positions_opened = 0
```

Pemeriksaan MT5 pasca-run read-only: terminal tetap connected; `orders_get()` = 0 dan `positions_get()` = 0. `ShadowRunner` memakai `NoExecution`, tidak punya adapter, dan `LiveEngine` dibangun dengan `out_path=None`, `emit=None`.

Frozen defaults tidak diubah: `StrategyConfig()` + `ReversalConfig()`, hash yang sama `b782c394aca8`. PATH A dan PATH C tetap disabled. Tidak ada optimasi/tuning; M15 structure, M5 confirmation; forming bar dikecualikan. `mt-info` strategy/config tidak disentuh dalam LIVE-2.

## 5. Tes regresi

Perintah (dari root `E:\mt5\mt-info`):

```powershell
python -m pytest python/tests/test_live_shadow_engine.py -q --override-ini="pythonpath=python"
# 18 passed

python -m pytest python/tests/test_shadow.py python/tests/test_live_pipeline.py python/tests/test_live_feed.py -q --override-ini="pythonpath=python"
# 295 passed, 1 skipped
```

Tes LIVE-1 dan shadow/live-feed regresi tetap PASS.

## 6. Kesimpulan

MT5 Windows connected dan feed real broker `XAUUSD.m` bekerja end-to-end: live history warm-up berhasil, forming M5 dikecualikan, lalu satu M5 close baru dinilai oleh frozen engine. Hasil run: **1 NO_TRADE (`no_reversal`), 0 SIGNAL**. Tidak ada order atau perubahan strategi. Run ini memverifikasi konektivitas dan pemrosesan satu candle close; bukan klaim performa atau edge.

## 7. Git

- Laporan berada di repo hasil `hasil-prompt-mt-info`.
- `E:\mt5\mt-info`: **jangan commit/push**; journal live berada lokal di repo itu dan tidak termasuk commit.
- Setelah menyimpan laporan, stop sesuai permintaan.
