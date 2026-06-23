# Alpha Lab — Blocker Report
**Date:** 2026-06-23  
**Author:** Jayavikram (Sys 3 — Alpha Lab, L2 Compute)  
**Sharing:** Surya, DB Team, Kavyansh (Feeder + Vault), Shreeya (QLoRA)  
**Sprint:** Week 5–6, June 23 – July 4 2026  

---

## Context

Alpha Lab (Sys 3) is the feature engineering engine for Singularity. It ingests raw market data from Sys2, computes a 58-column feature vector per OPTION_CHAIN snapshot, and hands the output Parquet to Vault for lineage stamping before QLoRA training and Helios backtesting.

The pipeline is **live and producing correct output** for all columns within Alpha Lab's control. However, 26 of 58 columns are either partially or entirely blocked by data availability, upstream data quality, or module integration gaps outside Alpha Lab's scope.

This document details every blocker, what is specifically needed, why it matters, and who owns the resolution.

---

## Current Feature Schema Completeness

**Schema:** FEATURE_SCHEMA v0.1.3 | **Total columns:** 58 | **VM data available:** 1.3 calendar days (2026-06-22 to 2026-06-23 only)

| Category | Columns | % of Schema |
|---|---|---|
| Fully Live — 100% populated | 32 | 55% |
| Partial — self-healing as data accumulates | 17 | 29% |
| Blocked — requires other module/team action | 9 | 16% |

### Fully Live (32 columns)

These are correctly computed and 100% populated from any single session of data.

| Group | Columns |
|---|---|
| Identity | `symbol`, `as_of_ts`, `underlying`, `option_type`, `strike`, `expiry_date` |
| Options | `log_moneyness`, `dte`, `dte_weekly`, `is_expiry_day` |
| Returns | `ret_1m`, `ret_5m`, `ret_15m`, `ret_30m`, `ret_1h` |
| RSI | `rsi_14_1m`, `rsi_14_5m` |
| MACD | `macd_hist_1m`, `macd_hist_5m` |
| Bollinger %B | `bollinger_pct_b_1m`, `bollinger_pct_b_5m` |
| ATR | `atr_14_1m`, `atr_14_5m` |
| ADX | `adx_14_1m`, `adx_14_5m` |
| Volatility | `realized_vol_1h`, `india_vix` |
| Forward Labels | `label_ret_1m_fwd`, `label_ret_5m_fwd`, `label_ret_15m_fwd`, `label_ret_30m_fwd`, `label_ret_1h_fwd` |

### Partial — Self-Healing (17 columns)

These columns require historical warmup bars from prior sessions. They are correctly implemented but produce NaN until sufficient Sys2 history exists. No code changes needed — they fill automatically as data accumulates.

| Column | Current Fill | Warmup Needed | Expected to Fill |
|---|---|---|---|
| `rsi_14_15m` | ~60% | 14 × 15m = 3.5h | Day 2 |
| `rsi_14_30m` | ~30% | 14 × 30m = 7h | Day 3 |
| `rsi_14_1h` | 0% | 14 × 1h across sessions | Day 3–4 |
| `macd_hist_15m` | 0% | 26 × 15m = 6.5h | Day 2–3 |
| `macd_hist_30m` | 0% | 26 × 30m = 13h | Day 3 |
| `macd_hist_1h` | 0% | 26 × 1h = 26h | Day 5–6 |
| `bollinger_pct_b_15m` | ~60% | 20 × 15m = 5h | Day 2 |
| `bollinger_pct_b_30m` | ~25% | 20 × 30m = 10h | Day 3 |
| `bollinger_pct_b_1h` | 0% | 20 × 1h = 20h | Day 4–5 |
| `atr_14_15m` | ~70% | 14 × 15m warmup | Day 2 |
| `atr_14_30m` | ~50% | 14 × 30m = 7h | Day 2–3 |
| `atr_14_1h` | ~12% | 14 × 1h across sessions | Day 3 |
| `adx_14_15m` | ~70% | 14 × 15m warmup | Day 2 |
| `adx_14_30m` | ~50% | 14 × 30m = 7h | Day 2–3 |
| `adx_14_1h` | ~12% | 14 × 1h across sessions | Day 3 |
| `vol_regime` | ~39% | Welford VIX z-score warmup | Day 10–15 |
| `iv_rank` | 0% | ≥ 5 trading days of IV history | Day 7 |

> All 17 of these columns are **blocked only by the absence of historical data in Sys2**. If the backfill (BLOCKER-1 below) is loaded, all 17 will immediately go to near-100%.

### Blocked by Other Modules (9 columns)

| Column | Fill % | Owner | Root Cause |
|---|---|---|---|
| `iv` | 79.4% | DB Team | Broker skips Greeks for deep OTM/ITM strikes |
| `delta` | 79.4% | DB Team | Same |
| `gamma` | 79.4% | DB Team | Same |
| `theta` | 79.4% | DB Team | Same |
| `vega` | 79.4% | DB Team | Same |
| `bid_ask_spread` | 0% | DB Team | `bid_p == ask_p == ltp` in OPTION_CHAIN feed — broker upstream issue |
| `order_book_imbalance` | 0% | DB Team | No per-contract `mkt.market_depth` for options chain |
| `volume_15s` | 0% | Feeder (Kavyansh) | Built in `feeder/option_microstructure.py`; not yet wired to Alpha Lab |
| `vwap_15s` | 0% | Feeder (Kavyansh) | Same |

---

## Blockers — Detailed

---

### BLOCKER-1 — Historical Data Backfill Not in Sys2

**Owner:** DB Team / Surya  
**Priority:** CRITICAL — blocks 26 columns and all downstream training  
**Impact:** QLoRA (Shreeya), Helios backtesting (Vishesh + Aarav), vol_regime, iv_rank

#### What is missing

Alpha Lab queried Sys2 for data from 2026-03-01 to 2026-06-23 (the sprint plan backfill target range). The actual data available:

| Feed | Instrument | Available From | Available To | Span |
|---|---|---|---|---|
| SPOT | `NSE_INDEX\|Nifty 50` | 2026-06-22 04:06 UTC | 2026-06-23 10:30 UTC | **1.3 days** |
| OPTION_CHAIN | `NSE_INDEX\|Nifty 50` | 2026-06-22 06:03 UTC | 2026-06-23 10:32 UTC | **1.2 days** |
| GREEKS | `mkt_market_greeks` | 2026-06-22 | 2026-06-23 | **1.2 days** |
| VIX | `NSE_INDEX\|India VIX` (IDX) | 2026-06-22 05:38 UTC | 2026-06-23 07:31 UTC | **2 daily snapshots** |

Nothing exists before 2026-06-22. The sprint plan states "With Upstox backfill now available…" — but this data is not yet in Sys2.

#### What is needed

| Data | Instrument Key | Type | Date Range | Priority |
|---|---|---|---|---|
| NIFTY SPOT ticks | `NSE_INDEX\|Nifty 50` | `SPOT` | 2026-03-01 → 2026-06-20 | Critical |
| NIFTY OPTION_CHAIN | `NSE_INDEX\|Nifty 50` | `OPTION_CHAIN` | 2026-03-01 → 2026-06-20 | Critical |
| Market Greeks | all NIFTY FO instruments | `GREEKS` | 2026-03-01 → 2026-06-20 | Critical |
| India VIX | `NSE_INDEX\|India VIX` | `IDX` or intraday | 2026-03-01 → 2026-06-20 | High |
| Market Depth | NIFTY FO chain instruments | `DEPTH` | 2026-03-01 → 2026-06-20 | Medium |

**Minimum viable:** 60 trading days (approximately 2026-03-01 to 2026-06-20) to satisfy:
- Sprint plan acceptance criterion: ≥ 60 trading days with < 5% null rate on Group 1 + MTF indicators
- vol_regime distribution with representation in all 3 buckets (≥ 5% each)
- iv_rank: ≥ 5 trading days minimum (ideally 30 days)
- QLoRA first training run: requires "at minimum 1 week of real data; 3+ months after backfill pipeline completes"

#### Why it matters — column impact

Without backfill, these columns will be null for the **foreseeable future**:

| Self-Healing Column | Requires | With 1.3 days | With 60 days |
|---|---|---|---|
| `macd_hist_1h` | 26h of 1h bars | 0% | 100% |
| `bollinger_pct_b_1h` | 20h of 1h bars | 0% | 100% |
| `rsi_14_1h` | 14h across sessions | 0% | 100% |
| `atr_14_1h` / `adx_14_1h` | 14 sessions | 12% | 100% |
| `vol_regime` | ~20 VIX snapshots | 39% | 100% |
| `iv_rank` | ≥ 5 trading days | 0% | 100% |

Without backfill, QLoRA trains on a dataset where 17 of 58 columns are majority-null — degrading signal quality for the alpha_signal and regime adapters.

#### Ask of DB Team / Surya

1. **ETA for Sys2 backfill load:** When will the 2026-03-01 to 2026-06-20 data be queryable via the Sys2 Fetch API at `http://100.69.129.52:8194`?
2. **Format confirmation:** Will the backfill data appear under the same endpoint (`/fetch_as_of`) and instrument keys (`NSE_INDEX|Nifty 50`) as the live data, so the Alpha Lab pipeline requires no code changes to consume it?
3. **GREEKS coverage in backfill:** Will the historical GREEKS data include per-strike IV for the full chain, or only ITM/near-ATM strikes (same gap as live)?

---

### BLOCKER-2 — Greeks Missing for Deep OTM/ITM Strikes

**Owner:** DB Team / Upstox broker upstream  
**Priority:** HIGH  
**Affected columns:** `iv`, `delta`, `gamma`, `theta`, `vega`  
**Current fill:** 79.4% — 20.6% of rows permanently null

#### What is happening

The Upstox WebSocket feed does not transmit Greeks (`iv`, `delta`, `gamma`, `theta`, `vega`) for strikes that are sufficiently far out-of-the-money or in-the-money. For NIFTY with ~80 strikes in the chain, approximately 16 strikes (8 deep OTM calls + 8 deep OTM puts) return null Greeks on every snapshot.

Confirmed on 9,440+ rows: null Greeks always co-occur on the same row — this is broker-side filtering, not a pipeline issue.

| Metric | Value |
|---|---|
| Total chain rows tested | ~71,600 |
| Rows with Greeks present | ~56,800 (79.4%) |
| Rows with null Greeks | ~14,800 (20.6%) |
| Null pattern | Always co-occur (iv/delta/gamma/theta/vega null together) |
| Cause | Broker (Upstox) does not send Greeks for deep OTM/ITM strikes |

#### Impact

- 20.6% of chain rows have incomplete feature vectors — reducing usable training data
- For QLoRA (Shreeya): rows with null Greeks are valid training rows for non-Greeks features but cannot train on iv/delta/gamma/theta/vega directly
- For Helios (Vishesh): strike selector must handle null Greeks gracefully

#### Ask of DB Team / Surya

1. **Server-side BSM computation:** Can Sys2 compute Black-Scholes Greeks server-side from `(spot_ltp, strike, iv, dte, risk_free_rate)` and fill the gap for strikes the broker skips? Even approximate values are better than null for ML training.
2. **If not:** Confirm that 79.4% is the expected ceiling and document in CONTRACTS.md so downstream teams (QLoRA, Helios) build null-handling accordingly.
3. **Chain width:** Was the OPTION_CHAIN subscription on 2026-06-22 narrower than on 2026-06-23? Greeks coverage varied significantly between the two sessions (47.6% vs 79.4%). If a wider subscription was added, that explains it — and historical data may have a narrower set.

---

### BLOCKER-3 — bid_ask_spread and order_book_imbalance: OPTION_CHAIN Data Quality

**Owner:** DB Team / Upstox broker upstream  
**Priority:** HIGH  
**Affected columns:** `bid_ask_spread`, `order_book_imbalance`  
**Current fill:** 0% (null-stubbed)

#### What is happening

The OPTION_CHAIN feed received via Sys2 carries `bid_p` and `ask_p` fields. These are confirmed to **mirror `ltp` exactly** for 100% of rows tested:

```
bid_p == ask_p == ltp   for all 9,440+ OPTION_CHAIN rows tested
```

This means `ask_p - bid_p` is always 0 — a misleading non-null value. Alpha Lab null-stubs `bid_ask_spread` rather than report 0, because 0 spread is factually incorrect and would corrupt model training.

`order_book_imbalance` requires Level 2 depth data (`bid_qty`, `ask_qty` at multiple price levels) which does not exist in the current OPTION_CHAIN feed.

#### What is needed

**For `bid_ask_spread`:**

Option A (preferred): A separate Sys2 table with real Level-1 best bid/best ask for FO instruments — something like `mkt.l1_quotes` with `instrument_type=CE` or `instrument_type=PE`, queried per option contract key.

Option B: Confirm whether a WebSocket subscription change can fix the `bid_p`/`ask_p` values in the existing OPTION_CHAIN feed (i.e., this is a subscription config issue rather than a broker limitation).

**For `order_book_imbalance`:**

`mkt.market_depth` rows for the full NIFTY options chain (~80 strikes × CE/PE = 160 contracts per timestamp). Currently only one contract key (`NSE_FO|62329`) is queryable for DEPTH.

| What | Instrument Type | Coverage Needed |
|---|---|---|
| L1 bid/ask per option contract | `CE`, `PE` or equivalent | All strikes in NIFTY chain |
| L2 depth (bid/ask qty at levels) | `DEPTH` | All strikes in NIFTY chain |

#### Ask of DB Team / Surya

1. **L1 bid/ask:** Does Sys2 have a table with real (non-LTP-echoed) bid and ask prices for NIFTY FO instruments? If yes, what is the instrument_type and table name?
2. **Root cause:** Is `bid_p == ask_p == ltp` an Upstox API limitation (they do not provide L1 for options), or a configuration issue in the Sys2 subscription?
3. **Chain DEPTH:** Can DEPTH data be queried for all instruments in a given expiry/underlying group, rather than only individual instrument keys?

---

### BLOCKER-4 — india_vix: Daily Snapshot Only, No Intraday Ticks

**Owner:** DB Team  
**Priority:** MEDIUM  
**Affected columns:** `india_vix` (populated but low resolution), `vol_regime` (quality degraded)  
**Current state:** 1 VIX value per calendar day

#### What is happening

`india_vix` is populated but Sys2 stores it as a **daily close snapshot** (`instrument_type=IDX`). There is only 1 row per day:

| Date | VIX Value |
|---|---|
| 2026-06-22 | 13.80 |
| 2026-06-23 | 12.87 |

Alpha Lab's `join_asof(backward)` propagates the same VIX value across all 25,000+ option rows in a single day. The column appears populated but carries no intraday information.

#### Impact on `vol_regime`

`vol_regime` uses a Welford expanding z-score on the VIX series. With only 1 unique VIX value per day, the z-score is always 0, mapping all rows to the same regime bucket (regime 3). This makes `vol_regime` effectively a constant — reducing its ML signal value to near zero.

All 39.2% of populated `vol_regime` rows have value = 3.

#### What is needed

Intraday VIX ticks at the same granularity as NIFTY SPOT — ideally 1-minute resolution, minimum per-session ticks (open/high/low/close per trading interval).

| What | Instrument Key | Type | Frequency Needed |
|---|---|---|---|
| Intraday India VIX | `NSE_INDEX\|India VIX` | `SPOT` or tick | Per-minute (or per-15s) |

#### Ask of DB Team / Surya

1. Does Sys2 capture intraday VIX ticks from Upstox, or only the EOD snapshot?
2. If intraday VIX is available, what is the correct `instrument_type` to query it (not `IDX`)?
3. If intraday VIX is not available from Upstox, is there an NSE-published intraday VIX series that could be ingested into Sys2?

---

### BLOCKER-5 — volume_15s and vwap_15s: Feeder Bundle Integration Not Wired

**Owner:** Kavyansh (Feeder Module — `feeder/option_microstructure.py`)  
**Priority:** HIGH  
**Affected columns:** `volume_15s`, `vwap_15s`  
**Current fill:** 0% (null-stubbed)

#### What is happening

`volume_15s` and `vwap_15s` are computed by the Feeder module in `feeder/option_microstructure.py` from `ltq` (last traded quantity) and `ltp` (last traded price) in `mkt_options_chain`. They are delivered to Alpha Lab via `FeatureInputBundle.options_15s`.

Alpha Lab currently calls Sys2 directly via `live_api.fetch_as_of()` and does **not** receive `FeatureInputBundle` from the Feeder. Until this wiring is done, both columns are null.

```
Current path:
  Alpha Lab → fetch_as_of() → Sys2 raw data → no volume_15s/vwap_15s

Required path:
  Feeder (option_microstructure.py) → FeatureInputBundle.options_15s
                                              ↓
                                       Alpha Lab receives bundle
                                       → volume_15s, vwap_15s populated
```

#### Column definitions

| Column | Formula | Source in Feeder |
|---|---|---|
| `volume_15s` | `sum(ltq)` over 15-second bar window for ATM strike | `mkt_options_chain.ltq` |
| `vwap_15s` | `sum(ltp × ltq) / sum(ltq)` over 15-second bar window | `mkt_options_chain.ltp`, `.ltq` |

#### Ask of Kavyansh (Feeder)

1. **Integration timeline:** When is `fetch_feature_input_bundle()` in `feeder/batch_fetcher.py` ready for Alpha Lab to call in a joint Integration Gate test?
2. **Join key:** What is the join key between `bundle.options_15s` rows and Alpha Lab's OPTION_CHAIN base frame? (e.g., `instrument_symbol` + `valid_time`)
3. **Schema of options_15s:** What columns does `bundle.options_15s` carry — specifically, are `volume_15s` and `vwap_15s` already renamed to those exact column names, or do they come under different field names?
4. **SNAP route fix:** Section 4.4 of the sprint plan mentions the SNAP route (`mkt_vix_snapshots`) has an `instrument_symbol` column mismatch that is blocking VIX data flow with `validate=True`. What is the ETA for this fix? It blocks the Integration Gate Step 1 pass condition.

---

### BLOCKER-6 — Vault Integration: Output Path and Contract

**Owner:** Kavyansh (Vault Module)  
**Priority:** HIGH  
**Affected:** All 58 columns — Vault cannot ingest Alpha Lab output until this is agreed

#### What is happening

Alpha Lab's `run_batch()` currently runs **locally** (on Jayavikram's development machine). The Polars DataFrame returned by `run_batch()` exists only in local memory. **Vault has no access to it.**

For the Integration Gate (Step 3) to pass, the pipeline must run on the VM and write output to a path Vault can read.

#### What needs to be agreed

| Item | Current State | Needed |
|---|---|---|
| VM output path | None defined | Agreed path (e.g., `/data/singularity/features/YYYY-MM-DD/features.parquet`) |
| `snapshot_id` in output | Not in 58-column output | Does Vault derive it from input rows, or should Alpha Lab pass it through? |
| Pandera schema | Not shared | Kavyansh to share `pandera_schema.py` class so Alpha Lab can validate before returning |
| Pipeline deployment | Code runs locally only | Alpha Lab `src/` needs to run on the VM for output to reach Vault |

#### Ask of Kavyansh (Vault)

1. **VM output path:** What exact filesystem path should Alpha Lab write daily Parquet files to? Proposed convention: `/data/singularity/features/YYYY-MM-DD/features.parquet`
2. **snapshot_id:** The sprint plan (section 4.2) says Vault derives `dataset_snapshot_id` from `df["dataset_snapshot_id"]` on the input rows. Alpha Lab's OPTION_CHAIN rows do carry `snapshot_id` in the raw data — should this be passed through as a column in the 58-column output, or does Vault handle this separately?
3. **Pandera schema:** Once `src/vault/pandera_schema.py` is written, share it with Alpha Lab so we can add a `validate()` call at the end of `run_batch()` for early error detection before handoff.

---

## Summary Table — All Asks

| # | Blocker | Owner | Columns Affected | Priority | Action Required |
|---|---|---|---|---|---|
| 1 | Historical backfill (2026-03-01 to 2026-06-20) not in Sys2 | DB Team / Surya | 17 partial columns + iv_rank + vol_regime | CRITICAL | Load backfill to Sys2; confirm ETA and query format |
| 2 | Greeks null for 20.6% of chain rows (deep OTM/ITM) | DB Team / Upstox | `iv`, `delta`, `gamma`, `theta`, `vega` | HIGH | Server-side BSM fill, OR confirm 79.4% is accepted ceiling |
| 3 | `bid_p == ask_p == ltp` in OPTION_CHAIN — no real L1 spread | DB Team / Upstox | `bid_ask_spread`, `order_book_imbalance` | HIGH | Provide real L1 bid/ask or confirm limitation |
| 4 | `india_vix` daily-only — no intraday ticks | DB Team | `india_vix` (quality), `vol_regime` | MEDIUM | Confirm if intraday VIX is available in Sys2 |
| 5 | Feeder bundle (`FeatureInputBundle.options_15s`) not wired | Kavyansh (Feeder) | `volume_15s`, `vwap_15s` | HIGH | Integration timeline + join key + column schema |
| 6 | Vault output path, snapshot_id, Pandera contract undefined | Kavyansh (Vault) | All 58 columns (ingestion blocked) | HIGH | Agree on VM path, snapshot_id format, share Pandera schema |

---

## Downstream Impact of Unresolved Blockers

| Downstream Module | What it needs from Alpha Lab | Blocked by |
|---|---|---|
| **QLoRA / Shreeya** | `data/production_features.parquet` — minimum 1 week, ideally 3 months | BLOCKER-1 (no backfill), BLOCKER-6 (no VM write path) |
| **Helios / Vishesh + Aarav** | `vol_regime` column from feature parquet for regime-stratified reporting | BLOCKER-1 (vol_regime only 39% with 1.3 days, needs 20+ days) |
| **Vault / Kavyansh** | Alpha Lab Parquet at agreed VM path with correct schema | BLOCKER-6 (path undefined, code running locally not on VM) |

---

## What Alpha Lab Has Already Delivered

For full transparency, the following are complete and working on Alpha Lab's side:

- 32 of 58 columns fully live and correctly computed
- Terminal NaN label rows dropped before output (Sprint plan 3.2 NON-NEGOTIABLE — done)
- Expired option rows (dte < 0) filtered before output
- `bid_ask_spread` null-stubbed correctly (was computing misleading 0 from LTP-echoed bid/ask)
- All 82 non-integration unit tests passing
- Pipeline runs end-to-end against live Sys2 API
- Full audit documented in `docs/PIPELINE_AUDIT_2026-06-23.md`

---

*Document maintained by Jayavikram. Last updated: 2026-06-23.*  
*Cross-reference: `docs/PIPELINE_AUDIT_2026-06-23.md`, `FEATURE_SCHEMA_PROPOSAL.md`, Sprint Week 5–6 plan.*
