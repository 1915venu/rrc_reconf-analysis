# DRB Reconfiguration Contents Prediction — Status & Results

**Researcher:** Venu (IIT Delhi)
**Period:** May 8 – May 14, 2026
**Status:** Pipeline complete and reproducible. Binary "reconf-will-occur" sub-task partially working. Full 25-output content prediction is currently **data-limited** (331 DRB-setup events across 9 captures).

---

## Table of Contents

1. [What this project does](#1-what-this-project-does)
2. [End-to-end pipeline](#2-end-to-end-pipeline)
3. [Important files & locations](#3-important-files--locations)
4. [The parser — how it works](#4-the-parser--how-it-works)
5. [The trainer — how it works](#5-the-trainer--how-it-works)
6. [Dataset schema (82 columns)](#6-dataset-schema-82-columns)
7. [Dataset — before vs after this week](#7-dataset--before-vs-after-this-week)
8. [Training results](#9-training-results)
9. [Top features by gain](#10-top-features-by-gain)
10. [Honest limitations](#11-honest-limitations)
11. [Recommendations for next steps](#12-recommendations-for-next-steps)

---

## 1. What this project does

**One-line goal:** Given the recent radio-layer telemetry from a UE (the phone), predict the parameter values inside the next RRC Reconfiguration message that the eNB will send, *before* it actually arrives.

**The setting:**

- In LTE, when the network (eNB / base station) wants to add a Data Radio Bearer (DRB) for the UE, it sends an `rrcConnectionReconfiguration` message over the air.
- This message carries a bundle of parameters: RLC mode (AM/UM), poll-retransmit timer, PDCP discard timer, logical-channel priority, etc.
- These parameters fundamentally shape how the UE buffers, transmits, and acknowledges data.

**The hypothesis:** The UE side has signals (MAC scheduling pattern, PHY SNR, measurement reports, PUCCH activity) that *foreshadow* an incoming reconfiguration. If we can predict the parameters 500 ms ahead, the UE could pre-warm its radio stack.

**The research framing:** *Predictive bearer management* driven entirely from UE-side observables, without any change to the eNB.

---

## 2. End-to-end pipeline

```
┌────────────────┐    ┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Android phone │    │ /dev/ttyUSB0    │    │   .mi2log file   │    │  parse_drb_*.py │    │   .csv / .xlsx  │
│  (DIAG mode)   │──▶│   (Qualcomm     │───▶│   (binary, but   │──▶│   (windowing +  │──▶│   (82 columns,   │
│  via USB cable │    │   DIAG stream)  │    │   MobileInsight  │    │   feature       │    │   ML-ready)     │
│                │    │                 │    │   decodable)     │    │   extraction)   │    │                 │
└────────────────┘    └─────────────────┘    └──────────────────┘    └─────────────────┘    └─────────────────┘
                                                                                                       │
                                                                                                       ▼
                                                                                          ┌─────────────────────┐
                                                                                          │ train_drb_*.py      │
                                                                                          │ (LightGBM, one      │
                                                                                          │  model per output)  │
                                                                                          └─────────────────────┘
                                                                                                       │
                                                                                                       ▼
                                                                                          ┌─────────────────────┐
                                                                                          │ models.pkl, metrics │
                                                                                          │ feature importance  │
                                                                                          └─────────────────────┘
```

**Three independent stages:**

| Stage | Script | Input | Output |
|-------|--------|-------|--------|
| Capture | `dm_capture.py` | Phone connected at `/dev/ttyUSB0` | `session.mi2log` (binary) |
| Parse  | `parse_drb_contents.py` | `.mi2log` | `batch_N.csv` (82 columns) |
| Train  | `train_drb_contents.py` | Combined parquet of all batches | `.pkl` of trained models + metrics |

---

## 3. Important files & locations



| What | Path |
|------|------|
| **Primary dataset (331 DRB-setup windows, every output meaningful)** | `/home/venu/Desktop/ueautomation/outputs/drb_features_drb_setup_only.xlsx` |
| **Weekly summary (this document)** | `/home/venu/Desktop/ueautomation/outputs/drb_weekly_summary_may14.md` |
| **9-sheet weekly Excel report** | `/home/venu/Desktop/ueautomation/outputs/drb_weekly_summary_may14.xlsx` |

### Scripts (the pipeline code)

| Stage | Path | Lines |
|-------|------|-------|
| Capture | `/home/venu/Downloads/MobileInsight-6.0.0/scripts/dm_capture.py` | ~140 |
| **Parser** | `/home/venu/Desktop/ueautomation/scripts/parse_drb_contents.py` | ~1,221 |
| Reparse all batches | `/home/venu/Desktop/ueautomation/scripts/reparse_all.sh` | ~40 |
| **Trainer** | `/home/venu/Desktop/ueautomation/scripts/train_drb_contents.py` | ~310 |

### Per-batch raw CSVs (`outputs/batches/`)

| File | Rows | Source capture |
|------|------|----------------|
| `batch_1_phase_d1.csv` | 6,262 | Phase_D_Batch_1 (24 MB) |
| `batch_2_phase_d2.csv` | 18,241 | Phase_D_Batch_2 (97 MB) — largest |
| `batch_4_download.csv` | 3,283 | Download scenario |
| `batch_5_browsing.csv` | 1,345 | Web browsing |
| `batch_6_voicecall.csv` | 755 | Voice call |
| `batch_7_datatoggle.csv` | 820 | Data toggle |
| `batch_8_verify.csv` | 1,156 | Verification |
| `batch_9_pucch_test.csv` | 3,561 | PUCCH test |
| `batch_10_phase_e2.csv` | 4,493 | Phase_E_Batch_2 (NEW this week) |

> `batch_3` does not exist — original Phase_C capture was empty, skipped.

### Combined dataset

| File | Rows × Cols | Description |
|------|-------------|-------------|
| `/home/venu/Desktop/ueautomation/outputs/drb_contents_combined.parquet` | 39,916 × 82 | All 9 batches concatenated. **The trainer loads this.** |

### Inspection Excel files

| File | Rows | Purpose |
|------|------|---------|
| `outputs/drb_features_drb_setup_only.xlsx` ★ | 331 | DRB-setup windows only — every output column has meaningful value |
| `outputs/drb_features_reconf_only.xlsx` | 6,867 | All windows with any reconf (meas / PHY / DRB / HO) |
| `outputs/drb_features_phase_d1_82col.xlsx` | 6,262 | Full Phase_D Batch 1 — time series of one capture |

### Model artifacts

| File | What it is |
|------|------------|
| `outputs/drb_contents_models.pkl` | Pickled `{"models", "encoders", "features"}` — load to predict |
| `outputs/model_out_*.txt` | One per output target, LightGBM's human-readable dump |
| `outputs/drb_contents_feature_importance.csv` | Gain-based ranking from `out_has_reconf` model |


## 4. The parser — how it works

**Path:** `/home/venu/Desktop/ueautomation/scripts/parse_drb_contents.py` (1,221 lines)

Internally it does 4 things:

1. **Replay the .mi2log file** via MobileInsight's `OfflineReplayer`. This streams decoded events one at a time into our `RawEventCollector` subclass.
2. **Bucket events into typed lists** — `mac_events`, `phy_serv_events`, `phy_cqi_events`, `rrc_events`, `nas_events`, `pucch_events`, plus state-tracking lists `rrc_drb_state_events` and `rrc_reconf_type_events`.
3. **Generate sliding windows** — every 50 ms, take a 500 ms-wide window ending at this point, aggregate all events that fall inside it into features (sums, means, counts, deltas).
4. **Look forward 500 ms** for each window to find the next RRC Reconfiguration. If one exists, extract its parameters from the XML into the 25 output columns. If not, all outputs are zero / "none".

### Key concepts

| Concept | Value | Why |
|---------|-------|-----|
| Window width | `WINDOW_MS = 500` | Long enough to aggregate multiple MAC TBs and meas reports |
| Stride | `STRIDE_MS = 50` | ~20 windows/sec; gives ~10× more training examples than non-overlapping windows |
| Label horizon | `HORIZON_MS = 500` | How far ahead we look for the reconfig label |

So at time *t*, the **input** is everything in `[t−500ms, t]`, and the **label** is "what reconfig (if any) happens in `[t, t+500ms]`."

### Key methods (where to look in the code)

| Method | Line | Job |
|--------|------|-----|
| `RawEventCollector` (class) | 374 | MobileInsight Analyzer subclass — receives live events |
| `_handle_rrc()` | 460 | Decodes RRC Reconfigs from XML, tracks RLC state, DRB activation timestamps |
| `_handle_mac()` | 539 | Extracts grant bytes, BSR triggers, HARQ IDs, padding ratio |
| `_handle_phy_serv()` | 577 | RSRP, RSRQ, SNR |
| `_handle_phy_csf()` | 595 | Wideband CQI |
| `_handle_pucch()` | 675 | SR count, ACK/NACK, Tx power |
| `extract_drb_params()` | 253 | Parses RRC XML → 25 `out_*` output columns |
| `build_windows()` | 746 | Sliding window aggregation, label lookup |

### Output extraction (the 25 output columns)

When a window's horizon contains an RRC Reconfiguration, `extract_drb_params(xml_root)` walks the Wireshark-style XML tree and pulls out:

- `out_has_reconf` — 1 (always, since we found one)
- `out_reconf_ts` — exact timestamp of the reconfig
- `out_num_drbs_added`, `out_has_drb_release` — DRB list info
- `out_drb_id`, `out_eps_bearer_id` — identity of the first DRB added
- `out_rlc_mode` — "AM" / "UM" / "TM" / "none"
- `out_rlc_t_poll_retx_ms`, `out_rlc_poll_pdu`, `out_rlc_poll_byte`, etc. — RLC AM-specific
- `out_pdcp_discard_timer_ms`, `out_pdcp_status_report`, `out_pdcp_rohc_enabled`
- `out_lc_priority`, `out_lc_pbr_kbps`, `out_lc_bsd_ms`, `out_lc_group`
- `out_has_meas_config`, `out_has_phy_config`, `out_has_mac_config`, `out_has_handover`

If no reconfig in the next 500 ms → all numeric outputs are 0, string outputs are "none".

---

## 5. The trainer — how it works

**Path:** `/home/venu/Desktop/ueautomation/scripts/train_drb_contents.py` (~310 lines)

What it does:

1. Loads `drb_contents_combined.parquet` (all batches merged).
2. One-hot encodes the `current_activity` column (Idle / Unknown / Browsing / VoiceCall etc.).
3. **Splits by batch_id** (not by random row) to avoid leakage from overlapping windows:
   - **Train:** batches 5, 6, 7, 8, 9, 10 (12,130 windows)
   - **Val:** batch 1 (6,262 windows)
   - **Test:** batches 2, 4 (21,524 windows)
4. For each of the 10 output targets, calls `_train_one()`:
   - **Binary** targets: `objective="binary"`, `scale_pos_weight = n_neg / n_pos` for imbalance, optimizes `average_precision`
   - **Multiclass** targets: `objective="multiclass"` with **inverse-frequency sample weights** so rare classes aren't ignored
   - Early stopping with patience=40 on the val set
5. Evaluates each model on val and test, prints AUC-PR / accuracy / confusion matrix.
6. Saves the model bundle, per-target text dumps, and a feature-importance CSV.

### The 10 output targets

```python
OUTPUT_TARGETS = [
    ("out_has_reconf",            "binary",     None),
    ("out_rlc_mode",              "multiclass", {"none":0, "AM":1, "UM":2}),
    ("out_rlc_t_poll_retx_ms",    "binary",     None),
    ("out_rlc_t_reordering_ms",   "multiclass", None),
    ("out_lc_priority",           "multiclass", None),
    ("out_lc_group",              "multiclass", None),
    ("out_pdcp_discard_timer_ms", "multiclass", None),
    ("out_pdcp_rohc_enabled",     "binary",     None),
    ("out_has_meas_config",       "binary",     None),
    ("out_has_mac_config",        "binary",     None),
]
```

**Why one model per target:** LightGBM doesn't natively support multi-output. Wrapping it would force every target to share hyperparameters, which is wrong — `out_has_reconf` is binary but `out_rlc_mode` is 3-class with heavy imbalance. One-model-per-target is the simplest correct approach and lets us see per-target metrics.

### Currently trained on 37 features

- 35 hardcoded columns (MAC × 13, PUCCH × 1, PHY × 5, RRC × 3, temporal × 10, NAS × 1, activity × 2)
- + 2 activity one-hots auto-generated (`activity_Idle`, `activity_Unknown`)
- = **37 features**

> The 9 `cur_rlc_*` columns are populated in the dataset (97.5% non-zero) but **not yet in FEATURE_COLS**. Adding them is the single highest-leverage pending improvement.

---

## 6. Dataset schema (82 columns)

Every row in the dataset represents one 500 ms window, with **57 input feature columns** and **25 output target columns**.

### Input features (57 columns)

| Group | Columns |
|-------|---------|
| **Meta** | `window_id`, `batch_id`, `window_ts` |
| **MAC** | `mac_event_count`, `total_grant_bytes`, `mean_grant_bytes`, `max_grant_bytes`, `grant_variance`, `sbsr_count`, `lbsr_count`, `phr_count`, `padding_ratio`, `unique_harq_ids`, `num_data_lcids_max`, `grant_bytes_delta`, `bsr_rate_change` |
| **RLC stats** (placeholder, currently zero) | `rlc_total_pdu`, `rlc_data_pdu`, `rlc_retx_pdu`, `rlc_poll_count`, `rlc_nack_count`, `rlc_ctrl_pdu` |
| **Current RLC config state** (from RRC XML tracking, now working) | `cur_rlc_mode`, `cur_t_poll_retx_ms`, `cur_poll_pdu`, `cur_poll_byte`, `cur_max_retx_threshold`, `cur_t_reordering_ms`, `cur_t_status_prohibit_ms`, `cur_sn_length`, `cur_num_active_drbs` |
| **PUCCH** | `pucch_sr_count`, `pucch_ack_count`, `pucch_nack_count`, `pucch_mean_tx_power` |
| **PHY** | `phy_rsrp_mean`, `phy_rsrq_mean`, `phy_snr_rx0_mean`, `phy_cqi_mean`, `phy_rsrp_delta` |
| **RRC** | `meas_report_count`, `reconf_in_window`, `time_since_last_reconf_ms` |
| **Temporal (NEW)** | `reconf_type_hist_0`, `reconf_type_hist_1`, `reconf_type_hist_2`, `meas_report_acceleration`, `inter_reconf_slope_ms`, `drb_3_age_ms`, `drb_4_age_ms`, `drb_5_age_ms`, `drb_6_age_ms`, `drb_7_age_ms` |
| **NAS** | `nas_esm_in_window` |
| **DRX** | `time_since_last_mac_ms` |
| **Activity** | `current_activity` (string), `time_into_activity_ms` |

### Output targets (25 columns)

| Group | Columns |
|-------|---------|
| **Label flag** | `out_has_reconf`, `out_reconf_ts`, `out_num_drbs_added`, `out_has_drb_release` |
| **DRB identity** | `out_drb_id`, `out_eps_bearer_id` |
| **RLC config** | `out_rlc_mode`, `out_rlc_t_poll_retx_ms`, `out_rlc_poll_pdu`, `out_rlc_poll_byte`, `out_rlc_max_retx`, `out_rlc_t_reordering_ms`, `out_rlc_t_status_prohibit_ms`, `out_rlc_sn_bits` |
| **PDCP config** | `out_pdcp_discard_timer_ms`, `out_pdcp_status_report`, `out_pdcp_rohc_enabled` |
| **LC config** | `out_lc_priority`, `out_lc_pbr_kbps`, `out_lc_bsd_ms`, `out_lc_group` |
| **Reconf flags** | `out_has_meas_config`, `out_has_phy_config`, `out_has_mac_config`, `out_has_handover` |

---

## 7. Dataset — before vs after this week

| Metric | Before (May 8) | After (May 14) |
|--------|----------------|----------------|
| Total batches | 8 | **9** (added Phase_E_Batch_2) |
| Total windows | 35,768 | **39,916** |
| Columns | 72 | **82** (+10 temporal features) |
| Reconf windows | 6,771 | **6,867** |
| DRB-setup windows | 283 | **331** |
| RLC modes (DRB only) | n/a | AM = 228, UM = 61, none = 42 |
| `cur_rlc_mode` non-zero | ~0% | **97.5%** |

---

### 7.1 Feature addition: 10 new temporal / rate-of-change features


| Feature | Why it should help |
|---------|--------------------|
| `reconf_type_hist_0/1/2` | "Recent burst of measurement reconfigs" is a strong precursor to handover. Exposing the last 3 reconf types lets the model spot escalation patterns. |
| `meas_report_acceleration` | 2nd derivative of meas report rate. UEs about to hand over emit a burst — *acceleration* matters more than count. |
| `inter_reconf_slope_ms` | When reconfigs cluster (gap shrinking), DRB add or HO is imminent. Linear-fit slope on last 3 gaps captures this. |
| `drb_3..7_age_ms` | Session context — a freshly-added DRB behaves differently from one that's been active for 60 seconds. |

**Empirical validation:** **3 of the 10 new features ranked in the top 5 by gain** in the trained model immediately, displacing all but one of the original RRC features. See section 10.

---

## 9. Training results

| Target | Task | Val | Test | Interpretation |
|--------|------|-----|------|----------------|
| `out_has_reconf` | binary | **AUC-PR 0.299** | **AUC-PR 0.231** | Will any reconfig occur in next 500 ms. Above random (~0.17). Modest signal. |
| `out_has_meas_config` | binary | AUC-PR 0.398 | AUC-PR 0.242 | Will reconfig contain `measConfig`. Slightly better. |
| `out_has_mac_config` | binary | AUC-PR 0.105 | AUC-PR 0.058 | Will reconfig contain MAC config. Weak. |
| `out_rlc_t_poll_retx_ms` | binary | AUC-PR 0.301 | AUC-PR 0.048 | Will `t-PollRetransmit` be set. Big val/test gap → overfit. |
| `out_pdcp_rohc_enabled` | binary | AUC-PR 0.002 | AUC-PR 0.002 | RoHC enabled (very rare). |
| `out_rlc_mode` | 3-class | 20.9% on DRB win | 32.8% on DRB win | AM / UM / none. **At random baseline.** |
| `out_rlc_t_reordering_ms` | multiclass | 20.9% | 32.8% | Correlated with `rlc_mode`. |
| `out_lc_priority` | multiclass | 26.9% | 24.6% | 1–16 priority. Data-limited. |
| `out_lc_group` | multiclass | 25.4% | 34.4% | Logical channel group. |
| `out_pdcp_discard_timer_ms` | multiclass | 26.9% | 29.5% | PDCP discardTimer. |

> Random baseline for 3-class multiclass is ~33%. Current accuracy hovers near baseline → parameter-prediction models are not learning meaningful signal yet, primarily because of low DRB-event volume (only 331 examples).

---

## 10. Top features by gain

From the `out_has_reconf` model:

| Rank | Feature | Gain |
|------|---------|------|
| 1 | `time_since_last_reconf_ms` | 47,204 |
| 2 | `time_since_last_mac_ms` | 23,838 |
| 3 | **`drb_3_age_ms`** (NEW) | 18,851 |
| 4 | **`inter_reconf_slope_ms`** (NEW) | 18,781 |
| 5 | **`drb_4_age_ms`** (NEW) | 16,665 |
| 6 | `max_grant_bytes` | 7,904 |
| 7 | `phy_snr_rx0_mean` | 7,624 |
| 8 | `total_grant_bytes` | 6,602 |
| 9 | `mean_grant_bytes` | 6,580 |
| 10 | `phy_rsrp_mean` | 6,387 |
| 11 | `reconf_in_window` | 6,025 |
| 12 | `pucch_sr_count` | 5,872 |
| 13 | `phy_rsrq_mean` | 5,636 |
| 14 | `time_into_activity_ms` | 3,851 |
| 15 | **`drb_5_age_ms`** (NEW) | 3,473 |

> **3 of the 10 new features ranked in the top 5** immediately, and a 4th made the top 15. The temporal / recency features are real signal.

**Caveat:** `time_since_last_reconf_ms` dominates by 2× the next feature — there's a risk the model is mostly learning the inter-event distribution rather than radio-state → reconfig content. This needs a leakage audit (train without it and measure).

---

## 11. Limitations

- **Data volume:** Only 331 DRB-setup events across 39,916 windows. Too few for reliable multi-output multiclass prediction (random baseline = 33% for 3-class).
- **Templated configs:** The eNB sends mostly canonical DRB parameter sets (carrier defaults). Most of the "prediction" achievable is a lookup table of standard configs, not a learned mapping from radio state.
- **Binary baseline:** `out_has_reconf` AUC-PR ~0.30 is below the threshold for publishable results. Need to push to **0.50+** before extending to full content prediction is credible.
- **PUCCH gaps:** `pucch_ack_count` and `pucch_nack_count` still zero — no Format 1A/1B traffic in any of the 9 captures, or the decoder isn't pulling the right field.
- **RLC dynamics gap:** Snapdragon 8 Gen 2 binary RLC Config format is undecodable; retx counts, NACK counts, poll events remain placeholder zeros. Currently bypassed via RRC-XML state tracking, but lose dynamics.
- **`cur_rlc_*` not in trainer yet:** The 9 columns are fixed and populated (97.5%) but the trainer doesn't use them. Highest-leverage pending one-line change.
- **Single device, single network:** All 9 captures from one phone on one operator. No cross-device validation.
- **Independent models:** 10 targets predicted independently — model can produce inconsistent joint predictions (e.g. `out_rlc_mode=UM` but `out_rlc_t_poll_retx_ms=80`).

---

## 12. Recommendations for next steps

| Option | Action | Effort | Expected outcome |
|--------|--------|--------|------------------|
| **A. Narrow scope** | Predict reconfig **type** (4-class: meas / DRB-add / PHY / HO) instead of full 25-output content | Low — uses existing dataset; 6,867 positives | Tractable; demonstrates new temporal features genuinely help; defensible result |
| **B. Add `cur_rlc_*` to FEATURE_COLS** | One-line change — add the 9 populated columns to the trainer's feature list and retrain | Trivial | Direct improvement on `out_rlc_mode` and `out_rlc_t_poll_retx_ms` |
| **C. More targeted captures** | Scripted bearer cycles: forced VoLTE drops, repeated data toggling, signal-attenuated handover triggers | Medium — ~1 week lab time | Only path to making 25-output content prediction statistically valid (>1000 DRB events) |
| **D. Fix binary baseline** | Push `out_has_reconf` AUC-PR from 0.30 → 0.50+. Feature pruning, leakage audit on `time_since_last_reconf_ms`, split-by-time instead of split-by-batch | Low–Medium | Credible baseline before reaching for content prediction |
| **E. Hierarchical model** | Two-stage: predict `has_reconf` first, then conditional on positives predict params using a model trained only on the 331 DRB-setup windows | Low | Concentrates learning capacity on the rare-event subset; may unstick multiclass models |


---
