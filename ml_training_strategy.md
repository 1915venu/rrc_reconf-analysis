# Smart RRC Reconfiguration Predictor — Complete ML Training Strategy

> **Goal:** Train an ML model that predicts RRC Reconfiguration messages **before** they arrive — both **when** they will occur and **what** they will contain.  
> **Dataset:** 13 batches across 3 phases from OnePlus 11R (Snapdragon 8 Gen 2) on Airtel 4G LTE, captured via MobileInsight with custom Qualcomm decoder.  
> **Date:** April 2026

---

## Table of Contents

1. [Dataset Assessment](#1-dataset-assessment)
2. [Data Pipeline: Raw Logs to Training Tensors](#2-data-pipeline-raw-logs-to-training-tensors)
3. [Feature Engineering: From 31 Columns to 30+ ML Features](#3-feature-engineering-from-31-columns-to-30-ml-features)
4. [DRX-Aware Design](#4-drx-aware-design)
5. [Model Architecture Recommendations](#5-model-architecture-recommendations)
6. [Predicting Reconfiguration Content (Stage 2)](#6-predicting-reconfiguration-content-stage-2)
7. [Class Imbalance Strategy](#7-class-imbalance-strategy)
8. [Evaluation Strategy](#8-evaluation-strategy)
9. [Generalization and Data Augmentation](#9-generalization-and-data-augmentation)
10. [Future Data Collection Recommendations](#10-future-data-collection-recommendations)
11. [Deployment Considerations](#11-deployment-considerations)
12. [Execution Roadmap](#12-execution-roadmap)

---

## 1. Dataset Assessment

### What We Have

| Component | Status | Details |
|-----------|--------|---------|
| MAC UL/DL transport blocks | Present | 31-column CSV with Grant_Bytes, Padding, BSR, PHR, HARQ, LCIDs |
| RRC message detection | Present | `Is_RRC_Reconf` flag + `RRC_Detail` string |
| Activity ground truth | Present | `labels.csv` with microsecond-precision START/STOP timestamps |
| Network/device snapshots | Present | `radio_props_*.txt`, `telephony_state_*.txt` |
| Phase diversity | Present | Baselines (A), idle-to-active (B), chaos/mixed (C) |

### Data Volume Summary

| Phase | Batches | Duration per Batch | Approx Reconfigs | Scenario Types |
|-------|---------|-------------------|-------------------|----------------|
| Phase A | 4 | 30-160 seconds | 24-103 per batch | VoiceCall, Browsing, DataToggle, UploadStress |
| Phase B | 4 | ~60 seconds | Varies | Idle-to-active transitions (Browsing) |
| Phase C | 5 | ~27 minutes | 140-860 per batch | Chaos: 19 mixed overlapping activities |
| **Total** | **13** | | **~3,471 positive events** | |

### Key Statistics from Analysis

- **Idle:** 597 CSV rows, 704 decoded packets, 180 KB raw
- **Active:** 11,772 CSV rows, 5,645 decoded packets, 3.6 MB raw
- **Measurement reports increase 8x** during active data
- **RRC reconfigurations increase 3x** during active data (30 idle vs 92 active per 60s)
- **MAC DRX config is consistent** across all scenarios: `onDuration=10sf, inactivity=100sf, longCycle=320ms, shortCycle=80ms`

### Verdict: Dataset is Strong for Initial Model Training

The 31 MAC/RLC columns contain all the raw ingredients needed. No re-decoding or parser changes required for the first model iteration. Feature engineering is done in Python on top of existing CSVs.

---

## 2. Data Pipeline: Raw Logs to Training Tensors

### 2.1 The Timestamp Problem (Critical First Step)

The CSVs are **event-based, not time-sampled**. Multiple rows share identical timestamps because the Qualcomm modem flushes diagnostic buffers in batches — a single `LTE_MAC_UL_Transport_Block` log packet contains multiple sub-samples that the parser flattens into separate rows with the same timestamp.

The data is also **heterogeneous by layer**: RRC rows have empty MAC fields (Grant_Bytes is empty), and MAC rows have empty RRC fields (Is_RRC_Reconf is empty). These are fundamentally different event streams sharing a single timeline.

### 2.2 Step 1 — Stream Separation and Cleaning

```
Raw CSV (31 columns, mixed event types)
    ├── MAC_UL stream (C-RNTI only)     ← filter out non-user-plane RNTI types
    ├── RRC stream                       ← reconfig events, measurement reports
    └── NAS stream                       ← attach/detach events
```

**Critical filter:** Only keep rows with `RNTI_Type = "C-RNTI"`. Rows with `(MI)Unknown`, `SPS-RNTI`, `P-RNTI`, `SI-RNTI`, `Temporary-C-RNTI` are not user-plane traffic. Approximately 50% of MAC rows are non-C-RNTI diagnostic noise.

**Parse the `LCIDs` string field** (e.g., `"L-BSR|Padding"`, `"3,4"`) into structured sub-fields for downstream feature computation.

### 2.3 Step 2 — Windowing Strategy

#### Why Not 100ms Windows

The originally proposed 100ms window is **too narrow** given the observed DRX configuration:

```
DRX Timeline (longCycle = 320ms):
├──AWAKE (10ms)──┤────────────SLEEPING (310ms)────────────┤
                  ↑                                        ↑
        onDurationTimer                           Next wake-up

100ms window during sleep: [   zero MAC events   ] ← empty feature vector, useless
```

During DRX long cycle, the modem is asleep for 310ms out of every 320ms. A 100ms window sitting inside that sleep period contains nothing.

#### Recommended: 250ms Windows with 50ms Stride

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| **Window size** | 250ms | Spans ~1 shortDRX cycle (80ms) + margin; guarantees at least 1 on-duration during active sessions |
| **Stride** | 50ms | 80% overlap; 20 samples/sec; sufficient temporal resolution without creating redundant near-identical samples |
| **Prediction horizon** | 500ms forward | Enough lead time for proactive action before reconfig arrives |
| **Yield** | ~32,400 windows per Phase C batch (~27 min) | Sufficient training volume |

```
Visual:
Timeline:  |--------250ms--------|
                |--------250ms--------|     (shifted 50ms)
                     |--------250ms--------|

DRX short cycle (80ms) fits within 250ms window
→ Most active-session windows capture at least 1 wake-up period
```

### 2.4 Step 3 — Label Assignment

**Target variable:** `reconfig_in_next_500ms` (binary)

For each window: scan forward 500ms from the window's end timestamp. If any row in that forward range has `Is_RRC_Reconf=1`, the label is 1 (positive).

**Critical subtlety:** Exclude the current window's reconfigurations from the label. If a window already contains a reconfig, that is a **feature** (`rrc_reconf_in_window`), not part of the forward-looking target. The model should predict *future* events, not detect current ones.

### 2.5 Step 4 — Activity Ground Truth Join

Join `labels.csv` to each window as features:

| Feature | Source | Description |
|---------|--------|-------------|
| `current_activity` | labels.csv | One-hot: VoiceCall, Browsing, DataToggle, UploadStress, Download, YouTube, FullStress, GuardBand, Idle |
| `time_into_activity_ms` | labels.csv + window timestamp | How far into the current activity |
| `time_since_activity_change_ms` | labels.csv + window timestamp | Time since last START/STOP transition |

This is critical because different activities have fundamentally different reconfig patterns:
- VoiceCall: ~0.24 reconf/sec
- Browsing: ~0.71 reconf/sec
- 100MB Download: ~0.66 reconf/sec

### 2.6 Step 5 — Train/Test Split

**Temporal batch-level splitting** (no random row splits — time-series data requires temporal boundaries to avoid look-ahead leakage):

| Split | Data | Purpose |
|-------|------|---------|
| **Train** | Phase A (4 batches) + Phase B (4 batches) + Phase C Batches 1-3 | Learn baseline signatures + mixed patterns |
| **Validation** | Phase C Batch 4 | Hyperparameter tuning |
| **Test** | Phase C Batch 5 | Final held-out evaluation |

**Cross-validation:** Leave-One-Batch-Out (LOBO) within training set — each fold holds out one complete batch as validation. This gives 11 folds with proper temporal boundaries.

---

## 3. Feature Engineering: From 31 Columns to 30+ ML Features

### 3.1 Key Principle

> All 30+ ML features are **computed from the existing 31 CSV columns** using Python/Pandas. No re-decoding or parser changes needed.

```
Existing CSV columns (raw ingredients)    →    ML Features (computed per 250ms window)
───────────────────────────────────────         ──────────────────────────────────────
Grant_Bytes                               →    total_grant_bytes, mean_grant_bytes, 
                                               max_grant_bytes, grant_variance,
                                               grant_bytes_delta_250ms
Has_SBSR, Has_LBSR                        →    sbsr_count, lbsr_count, bsr_rate_change
Timestamp, Is_RRC_Reconf                  →    time_since_last_reconf_ms, 
                                               inter_reconf_interval_trend
LCIDs                                     →    active_bearer_bitmap, data_lcid_bitmap
```

### 3.2 Per-Window Aggregated Features (from MAC_UL C-RNTI events)

| Feature | Computation | Source Column(s) |
|---------|------------|------------------|
| `mac_event_count` | count of MAC UL events in window | row count |
| `total_grant_bytes` | sum(Grant_Bytes) in window | `Grant_Bytes` |
| `mean_grant_bytes` | mean(Grant_Bytes) in window | `Grant_Bytes` |
| `max_grant_bytes` | max(Grant_Bytes) in window | `Grant_Bytes` |
| `grant_variance` | variance(Grant_Bytes) — burstiness indicator | `Grant_Bytes` |
| `total_padding_bytes` | sum(Padding_Bytes) in window | `Padding_Bytes` |
| `padding_ratio` | Padding_Bytes / Grant_Bytes — utilization efficiency | `Padding_Bytes`, `Grant_Bytes` |
| `sbsr_count` | count(Has_SBSR=1) | `Has_SBSR` |
| `lbsr_count` | count(Has_LBSR=1) | `Has_LBSR` |
| `phr_count` | count(Has_PHR=1) — Power Headroom Reports | `Has_PHR` |
| `unique_harq_ids` | count distinct HARQ IDs — parallel HARQ utilization | `HARQ_ID` |
| `data_lcid_bitmap` | 5-bit vector: which LCIDs 3,4,5,6,7 are active | `LCIDs` |
| `sig_lcid_count` | count of SRB1/SRB2 signaling events | `LCIDs` |
| `num_data_lcids_max` | max Num_Data_LCIDs in window | `Num_Data_LCIDs` |

### 3.3 RRC-Derived Features (per window)

| Feature | Computation | Source Column(s) |
|---------|------------|------------------|
| `measurement_report_count` | count of measurementReport messages | `Layer`, `Message` |
| `paging_count` | count of paging messages | `Layer`, `Message` |
| `sib_count` | count of system information messages | `Layer`, `Message` |
| `rrc_reconf_in_window` | binary: was there a reconfiguration in this window | `Is_RRC_Reconf` |
| `time_since_last_reconf_ms` | elapsed ms since last Is_RRC_Reconf=1 | `Timestamp`, `Is_RRC_Reconf` |

### 3.4 Temporal Rate-of-Change Features (MOST IMPORTANT ADDITION)

These capture the **trajectory** of network state. The eNB triggers reconfigs based on **changing conditions**, not absolute thresholds.

| Feature | Computation | Why It Matters |
|---------|------------|----------------|
| `grant_bytes_delta_250ms` | (current window grant sum) - (previous window grant sum) | Sharp positive delta = traffic ramp-up = #1 predictor of SCell activation. Example: delta jumps from 500 to 5000 = download starting, eNB will activate carrier aggregation |
| `measurement_report_acceleration` | 2nd derivative of measurement report rate over trailing 1s | eNB triggers reconfigs on *changing* signal conditions. A burst of 3 reports in 500ms after silence almost certainly precedes a handover |
| `bsr_rate_change` | BSR event rate in current window vs trailing 1s average | Rising BSR = UE buffer filling faster than eNB is granting, eNB will reconfigure (add bearers, activate SCells) |
| `inter_reconf_interval_trend` | last 3 inter-reconfiguration gaps as feature vector | Reconfigs cluster: handovers trigger bursts of 3-5 rapid reconfigs within 500ms. Shrinking gaps = mid-burst, more coming |
| `sig_to_data_ratio_trend` | trend of signaling-to-data byte ratio | Rising = increasing control plane activity, precedes reconfigs |
| `mac_burst_indicator` | binary: >10x increase in MAC event rate in last 2s vs preceding 5s | Captures idle-to-active transition that Phase B specifically profiled |

#### Concrete Example: grant_bytes_delta

```
Window N-1:  total_grant_bytes = 500   (low traffic, idle browsing)
Window N:    total_grant_bytes = 5000  (10x jump — download started!)
→ grant_bytes_delta = +4500

The eNB sees this demand spike too, and will likely respond with:
  rrcConnectionReconfiguration { sCellToAddModList: [...] }
The delta PREDICTS the reconfig; the absolute value (5000) does not.
```

#### Concrete Example: measurement_report_acceleration

```
Last 2 seconds:  0 measurement reports (signal stable)
Last 500ms:      3 measurement reports  (signal changing!)
→ acceleration = HIGH

What happened: user moved, serving cell RSRP dropped below A3 threshold,
UE fired reports to eNB. eNB will respond with handover command.
→ Prediction: handover reconfiguration within 500ms ← VERY LIKELY
```

### 3.5 DRX-Aware Features (3GPP Domain-Specific)

| Feature | Computation | Why It Matters |
|---------|------------|----------------|
| `drx_cycle_phase` | Enum {likely_awake, short_cycle_sleep, long_cycle_sleep} computed from known DRX config and time since last MAC event | Reconfig CANNOT arrive during DRX sleep unless paging wakes UE. Encodes protocol constraint directly |
| `time_since_last_mac_activity_ms` | ms since any MAC event | >100ms (inactivity timer) = entered DRX. Reconfigs after DRX entry are qualitatively different (typically MeasConfig-only) vs during active scheduling |

### 3.6 Bearer State Features

| Feature | Computation | Why It Matters |
|---------|------------|----------------|
| `active_bearer_bitmap` | 5-bit vector: which DRBs (3,4,5,6,7) had traffic in last 1s | More informative than count: DRB-5 alone = VoLTE; DRB-3+4 = dual-bearer data. Specific combinations indicate specific services |
| `bearer_activation_recency` | ms since each bearer first appeared | Newly activated bearer (e.g., DRB-5 for voice) triggers follow-up reconfigs for measurement config adjustments |

### 3.7 Activity-Derived Features (from labels.csv join)

| Feature | Source |
|---------|--------|
| `current_activity` | One-hot encoding of activity in progress |
| `time_into_activity_ms` | Elapsed time since current activity started |
| `time_since_activity_change_ms` | Time since last START/STOP transition |

### 3.8 What's Missing — PHY Layer

| Available? | Feature | Impact |
|-----------|---------|--------|
| **Not captured** | RSRP (serving cell signal strength) | Would improve handover prediction by ~10-15% |
| **Not captured** | RSRP (neighbor cells) | Would enable handover TARGET prediction |
| **Not captured** | RSRQ (signal quality) | Would improve SCell activation prediction |
| **Not captured** | CQI (Channel Quality Indicator) | Would enable PHY_Config reconfig prediction |
| **Not captured** | SINR/SNR | Would improve overall prediction accuracy |

**Impact assessment:**
- For **WHEN** prediction (binary): ~5-10% AUC-PR improvement if added. **Model works well without it.**
- For **WHAT** prediction (content): Much more critical. CQI predicts PHY_Config, RSRP trajectory predicts handover target PCI.
- **Recommendation:** Add in future Phase D captures (see Section 10).

---

## 4. DRX-Aware Design

### 4.1 What is DRX?

DRX (Discontinuous Reception) is the modem's power-saving sleep schedule. The eNB configures the UE with a DRX cycle during RRC connection setup. The UE alternates between sleeping (radio off, saving battery) and waking (listening for scheduling grants).

### 4.2 Observed DRX Configuration (Consistent Across All Scenarios)

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `onDurationTimer` | 10 subframes (10ms) | Modem stays awake for 10ms each cycle |
| `drx-InactivityTimer` | 100 subframes (100ms) | After last PDCCH activity, stay awake 100ms more |
| `drx-RetransmissionTimer` | 8 subframes (8ms) | Wait for retransmissions |
| `longDRX-CycleStartOffset` | 320ms | Long sleep cycle duration |
| `shortDRX-Cycle` | 80ms | Short sleep cycle duration |
| `drxShortCycleTimer` | 1 (80ms) | Duration in short cycle before transitioning to long |

### 4.3 DRX State Machine

```
                    ┌──────────────┐
                    │   ACTIVE     │  ← Continuous scheduling, modem always awake
                    │ (inactivity  │    MAC events every 1-10ms
                    │  timer runs) │
                    └──────┬───────┘
                           │ No PDCCH for 100ms (inactivity timer expires)
                           ▼
                    ┌──────────────┐
                    │  SHORT DRX   │  ← Wake every 80ms for 10ms
                    │  (80ms cycle)│    Sparse MAC events
                    └──────┬───────┘
                           │ After 1 short cycle (80ms) with no activity
                           ▼
                    ┌──────────────┐
                    │  LONG DRX    │  ← Wake every 320ms for 10ms
                    │ (320ms cycle)│    Very sparse MAC events (10ms awake / 310ms asleep)
                    └──────────────┘
                           │
                           │ Paging or new UL data arrives
                           ▼
                    ┌──────────────┐
                    │   ACTIVE     │  ← Back to continuous scheduling
                    └──────────────┘
```

### 4.4 Why This Matters for ML

1. **Empty windows are NOT missing data** — they're the modem sleeping. The model must distinguish "no data because sleeping" from "no data because idle but awake."
2. **Reconfigs CANNOT arrive during DRX sleep** — the UE isn't listening. The eNB either pages the UE first (waking it) or waits for the next on-duration. The `drx_cycle_phase` feature encodes this constraint.
3. **The DRX config is the same across all scenarios** — this is a stable foundation. The model doesn't need to learn different DRX behaviors for different activities.
4. **Post-DRX wake-up is a strong predictor** — when the modem transitions from long DRX back to active (triggered by new data), a burst of reconfigurations often follows for measurement setup and bearer configuration.

---

## 5. Model Architecture Recommendations

### 5.1 Training Progression (In Order)

```
Step 1: LightGBM (baseline)      →  Evaluate  →  Feature importance analysis
Step 2: LightGBM + temporal features  →  Evaluate improvement
Step 3: TCN (sequence model)     →  Compare with LightGBM
Step 4: Ensemble (LightGBM + TCN)  →  Final model
```

### 5.2 Tier 1: LightGBM Baseline (START HERE)

**Why LightGBM, not XGBoost:**
- Handles sparse features natively (many windows have zero MAC events during DRX)
- Faster training
- Handles categorical features (current_activity, RNTI_Type) without one-hot encoding overhead

**Recommended Configuration:**

```python
params = {
    'objective': 'binary',
    'metric': 'average_precision',  # AUC-PR
    'is_unbalance': True,           # handles class imbalance natively
    'num_leaves': 63,               # start here, tune up to 127
    'feature_fraction': 0.7,        # regularization for limited data
    'bagging_fraction': 0.8,
    'bagging_freq': 5,
    'learning_rate': 0.05,
    'num_iterations': 500,
    'early_stopping_rounds': 50,
    'verbose': -1,
}
```

**Why start here:**
- Trains in **seconds** (vs hours for neural networks)
- Gives **feature importance** — tells you which of your 30+ features actually matter
- Handles missing values and class imbalance natively
- **Expected: ~0.75-0.85 AUC-PR**
- ~200KB model, <1ms inference on ARM

### 5.3 Tier 2: Temporal Convolutional Network (TCN)

**Why TCN, NOT LSTM/GRU:**

| Property | LSTM | TCN |
|----------|------|-----|
| Data needed to avoid overfitting | >10,000 positive events | ~3,000 can work |
| Training speed | Slow (sequential) | Fast (parallel convolutions) |
| Overfitting risk at our data size | High | Lower |
| On-device inference | Complex (hidden state) | Simple (pure convolutions) |
| Temporal pattern capture | Good but needs more data | Good with dilated convolutions |
| Our positive event count | 3,471 | 3,471 |

**Architecture:**

```
Input: [window_t-19, window_t-18, ..., window_t-1, window_t]
       └── last 20 windows = 5 seconds of context ──┘

       ┌─────────────────────────────┐
       │  Dilated Causal Conv Layer 1 │  dilation=1, kernel=3
       │  (sees neighboring windows)  │  32 hidden channels
       └──────────────┬──────────────┘
                      ▼
       ┌─────────────────────────────┐
       │  Dilated Causal Conv Layer 2 │  dilation=2, kernel=3
       │  (sees windows 2 apart)      │  32 hidden channels
       └──────────────┬──────────────┘
                      ▼
       ┌─────────────────────────────┐
       │  Dilated Causal Conv Layer 3 │  dilation=4, kernel=3
       │  (sees windows 4 apart)      │  32 hidden channels
       └──────────────┬──────────────┘
                      ▼
       ┌─────────────────────────────┐
       │  Dropout (0.3) + Sigmoid     │
       └──────────────┬──────────────┘
                      ▼
       P(reconfig in next 500ms) = 0.82
```

**Effective receptive field:** 15 steps = 3.75 seconds (covers DRX cycles and reconfig burst patterns)

**Why dilated convolutions matter:**
- Dilation=1: captures 50ms-scale patterns (individual MAC events)
- Dilation=2: captures 100-200ms patterns (DRX short cycles)
- Dilation=4: captures 400ms-1s patterns (activity transitions, reconfig bursts)

**Specs:** ~50K parameters, ~200KB float16, 2-4ms inference on ARM via ONNX Runtime.

### 5.4 Tier 3: Transformer — Only When Data Grows 5-10x

**Premature at current data volume.** Revisit when dataset reaches 15,000+ positive events across multiple devices/networks.

If pursued later:
- Small encoder-only: 2 layers, 4 attention heads, embedding dim 64
- Input: 40 steps (10 seconds of context)
- Sinusoidal position encoding aligned to DRX cycle periodicity

### 5.5 Ensemble Strategy

LightGBM and TCN make **complementary errors:**
- LightGBM excels at: individual feature thresholds ("if BSR count > 3 AND grant_delta > 2000 → predict reconfig")
- TCN excels at: temporal patterns ("this sequence of increasing grants matches the ramp-up pattern before SCell activation")

**Ensemble:** `P_final = alpha * P_lightgbm + (1 - alpha) * P_tcn` where alpha is learned on validation set.

---

## 6. Predicting Reconfiguration Content (Stage 2)

### 6.1 The Problem

Stage 1 predicts **WHEN** (binary: reconfig yes/no in next 500ms). Stage 2 predicts **WHAT** the reconfiguration will contain.

### 6.2 Feasibility by Reconfiguration Type

| Reconfig Type | Feasibility | Rationale | Data Needed |
|---------------|-------------|-----------|-------------|
| **MeasConfig** | **High** | ~75% of all reconfigs. Semi-periodic, follows connection setup patterns. Predictable from timing alone | Current data sufficient |
| **MeasObj_EUTRA additions** | **High** | Added shortly after connection/handover. Highly correlated with prior reconfig type | Current data sufficient |
| **Handover (occurrence)** | **Medium-High** | Predictable from measurement report bursts | Current data sufficient |
| **Handover (target PCI)** | **Low** | Requires neighbor cell RSRP (PHY layer) | Needs PHY capture (0xB193) |
| **DRB_Setup** | **Medium** | Follows NAS ESM procedures (captured) | Current data sufficient |
| **DRB_Release** | **Medium** | Release timing less predictable | Current data marginal |
| **SCell_Config** | **Medium** | Activation correlates with sustained high throughput | Current data sufficient |
| **PHY_Config** | **Low** | Requires CQI/rank indicator data | Needs PHY capture |
| **MAC_Config (DRX changes)** | **Low** | DRX config never changes in our data | Not predictable |

### 6.3 Approach: Hierarchical Multi-Label Classification

```
Input features
     │
     ▼
┌─────────────┐
│   Stage 1   │  Binary: will a reconfig happen in next 500ms?
│  (LightGBM  │  P(reconfig) > threshold?
│   or TCN)   │     │
└─────────────┘     │
                    │ YES (Stage 1 fires)
                    ▼
              ┌─────────────┐
              │   Stage 2   │  Multi-label: 7 sigmoid outputs
              │ (separate   │  ├── Target_MeasConfig       (0/1)
              │  model)     │  ├── Target_Handover          (0/1)
              └─────────────┘  ├── Target_DRB_Setup         (0/1)
                               ├── Target_DRB_Release       (0/1)
                               ├── Target_SCell_Config      (0/1)
                               ├── Target_PHY_Config        (0/1)
                               └── Target_MAC_Config        (0/1)
```

**Stage 2 uses focal loss** (alpha=0.25, gamma=2.0) for rare types like Handover (~4.6% of reconfigs).

### 6.4 Reconfig Type History Feature

The `reconf_type_history` feature encodes the last 3 reconfig types as a categorical sequence. Certain sequences are stereotyped:

```
Step 1: MeasConfig|MeasObj_EUTRA       ← "measure these neighbors"
Step 2: Handover|TargetPCI=87          ← "switch to cell 87"
Step 3: MeasConfig|MeasObj_EUTRA|PHY|MAC  ← "reconfigure after handover"
```

If the model sees Step 1, it can predict Step 2 is coming. After Step 2, Step 3 is almost certain.

---

## 7. Class Imbalance Strategy

### 7.1 Quantifying the Imbalance

With 250ms windows at 50ms stride over Phase C Batch 1 (~27 minutes):

```
Total windows:    ~32,400
Positive windows: ~3,000-4,000  (each reconfig creates ~10 positive windows
                                 within the 500ms horizon, but many cluster)
Ratio:            ~8:1 to 10:1  negative:positive
```

**This is moderate imbalance — very manageable.**

### 7.2 What TO Use

| Strategy | Implementation | Why |
|----------|---------------|-----|
| **Sample weighting** | LightGBM: `scale_pos_weight = neg_count / pos_count`; TCN: `pos_weight` in `BCEWithLogitsLoss` | Tells model "positive samples are worth 8-10x more" |
| **Temporal-aware negative downsampling** | Drop 50% of "obviously idle" negatives (no MAC activity >1s). Keep 100% of negatives within 2s of a positive | "Near-miss" negatives are the hardest and most informative training examples |
| **Focal loss (TCN)** | alpha=0.25, gamma=2.0 | Auto-downweights easy negatives (deep idle), focuses learning on hard cases (200-400ms before a reconfig where features are transitioning) |

### 7.3 What NOT to Use

| Avoid | Reason |
|-------|--------|
| **SMOTE** | Creates synthetic samples by interpolating between positives. On time-series data, this produces samples that blend "just before reconfig" with "5 seconds after reconfig" — **phantom patterns that never existed in reality** |
| **Strict 1:1 balancing** | Destroys base-rate information. Model can't learn that reconfigs are rare events — outputs uncalibrated probabilities |
| **BalancedRandomForest** | Trains separate models on balanced subsets, destroying temporal continuity within each subset |

---

## 8. Evaluation Strategy

### 8.1 Primary Metric: AUC-PR (NOT AUC-ROC)

**Why not AUC-ROC?** In imbalanced settings, AUC-ROC is misleadingly high because a model that never predicts positive still achieves a high True Negative Rate. AUC-PR directly measures the tradeoff that matters:
- **Precision:** When the model says "reconfig coming," how often is it right?
- **Recall:** Of all actual reconfigs, how many did the model catch?

### 8.2 Operating Point Selection

| Use Case | Favor | Acceptable | Reasoning |
|----------|-------|------------|-----------|
| Proactive resource prep (pre-allocate buffers, pre-compute handover params) | **Recall 0.85+** | Precision 0.5+ | False positive = minor wasted compute. False negative = defeats the purpose |
| Power optimization (keep modem awake vs enter DRX) | **Precision 0.8+** | Recall 0.6+ | False positive = wasted battery. False negative = normal DRX wakeup handles it |

### 8.3 Per-Scenario Evaluation (Critical)

Report metrics **separately** for each scenario type:

| Scenario | What to Check |
|----------|---------------|
| Idle/GuardBand | Baseline false positive rate — should be near zero |
| VoiceCall | Reconfig prediction during VoLTE (lower MAC activity, focused on bearer management) |
| Browsing | Bursty pattern, frequent measurement reconfigs |
| UploadStress | Sustained high UL, SCell-related reconfigs |
| Download | High throughput, carrier aggregation changes |
| Mixed/Chaos | The real test — overlapping activities |

**Additional critical metrics:**
- **Handover prediction recall specifically** — handovers are highest-impact reconfigs (momentary service interruption). If model catches 90% of MeasConfig but only 50% of handovers, that's a significant weakness
- **Prediction lead time distribution** — histogram of how far in advance correct predictions fire. If all correct predictions are <50ms before the event, the prediction is useless for proactive action

### 8.4 Cross-Validation Protocol

**Leave-One-Batch-Out (LOBO)** within Phase C:
- 5-fold CV where each fold holds out one Phase C batch entirely
- Report mean and standard deviation across folds
- High variance = model overfitting to batch-specific patterns

**Cross-Phase evaluation:**
- Train on Phase A+B only → test on Phase C: do clean baselines transfer to chaotic mixed workloads?
- Train on Phase C only → test on Phase A+B: does chaos training generalize to clean scenarios?

---

## 9. Generalization and Data Augmentation

### 9.1 Current Generalization Limitation

| Dimension | Current Coverage | Risk |
|-----------|-----------------|------|
| Device | 1 (OnePlus 11R, Snapdragon 8 Gen 2) | Model learns this specific modem's behavior |
| Chipset firmware | 1 (Q_V1_P14) | BSR/grant patterns are firmware-dependent |
| Network operator | 1 (Airtel India, MCC=404, MNC=10) | Model learns Airtel's eNB algorithms (likely Ericsson/Nokia) |
| Location/Cell site | 1 (PCIs 84/86/87) | Model learns this cell's configuration and neighbor topology |
| Frequency band | 1 (Band 1 primary + Band 8 secondary) | Carrier aggregation behavior is band-dependent |

**Any model trained on this data will be specific to this device + network + location combination.** This is expected and acceptable for the first iteration.

### 9.2 Safe Augmentation for Current Data

| Technique | Implementation | Safety |
|-----------|---------------|--------|
| **Gaussian noise** | Add N(0, 0.05 * feature_std) to continuous features during training | Safe — simulates minor measurement variations |
| **Random time-warping** | Stretch/compress feature sequences by 0.9-1.1x | Safe — simulates slightly different scheduling delays |
| **Feature normalization for transferability** | Express Grant_Bytes as fraction of theoretical max (not absolute). Express timing relative to DRX config ("2.5 long DRX cycles" not "800ms") | Makes features more portable to different network configs |

### 9.3 Transfer Learning for New Data

When new data from different networks/devices arrives:

```
┌─────────────────────┐
│  Base Model          │  ← Trained on current Airtel data (frozen weights)
│  P_base(reconfig)    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Adaptation Model    │  ← Small model trained on new network's data
│  P_adapted(reconfig) │
└──────────┬──────────┘
           │
           ▼
P_final = alpha * P_base + (1-alpha) * P_adapted
           └── alpha learned per-network on validation set
```

---

## 10. Future Data Collection Recommendations

### 10.1 Log Codes to ADD in Future Captures

#### Priority 1 — PHY Measurements (BIGGEST impact, add FIRST)

```python
# Add to MobileInsight capture script:
src.enable_log("LTE_PHY_Connected_Mode_Intra_Freq_Meas")  # 0xB179
src.enable_log("LTE_PHY_Serv_Cell_Measurement")            # 0xB193
```

**New fields this provides:**

| Field | Type | Why It Matters |
|-------|------|----------------|
| `RSRP_serving` | float (dBm) | Dropping RSRP is the primary trigger for handover |
| `RSRP_neighbors` | float[] (dBm) | Which neighbor is strongest → predicts handover TARGET PCI |
| `RSRQ` | float (dB) | Signal quality degradation → triggers SCell changes |
| `SINR / SNR` | float (dB) | Noise environment → affects throughput, triggers PHY reconfigs |
| `CQI` | int (0-15) | Channel Quality Indicator → directly drives MCS/MIMO decisions |

#### Priority 2 — Structured RRC Content Parsing

Currently `RRC_Detail` is a pipe-delimited string (e.g., `"MeasConfig|MeasObj_EUTRA"`). Add structured boolean columns to the parser:

```python
# In RRC parser callback, extract from decoded dict:
fields_to_add = {
    'has_measConfig':                    bool,   # measurement config present?
    'has_mobilityControlInfo':           bool,   # handover command present?
    'has_radioResourceConfigDedicated':  bool,   # bearer/PHY changes?
    'has_sCellToAddModList':             bool,   # carrier aggregation changes?
    'num_measObjectToAddMod':            int,    # measurement objects added count
    'num_reportConfigToAddMod':          int,    # report configs added count
    'target_physCellId':                 int,    # handover target PCI (if handover)
}
```

#### Priority 3 — RLC Statistics (Nice to Have)

```python
src.enable_log("LTE_RLC_UL_AM_All_PDU")   # 0xB191
src.enable_log("LTE_RLC_DL_AM_All_PDU")   # 0xB192
```

| Field | Why |
|-------|-----|
| `rlc_retransmission_count` | High retransmissions = bad channel → reconfig likely |
| `rlc_throughput_per_bearer` | Per-bearer throughput trend |

#### Skip These (Diminishing Returns)

| Skip | Why |
|------|-----|
| NAS layer details | Too slow (seconds-scale), doesn't help 500ms prediction window |
| PDCP layer | Mostly duplicates RLC info at this granularity |
| System Information (SIBs) | Broadcasts, not UE-specific, won't predict reconfigs |
| Full RRC ASN.1 decode | Massive overhead, the structured booleans above capture what matters |

### 10.2 Scenario Diversity for Future Iterations

**Ranked by impact on model generalization:**

| Priority | What to Capture | Why | Effort |
|----------|----------------|-----|--------|
| **1 (Highest)** | Same location, different times of day (5 sessions: morning rush, midday, evening peak, late night, weekend) | Network load varies by time → teaches model to handle different eNB scheduling pressure | Low (same setup, just different times) |
| **2** | Different cell sites on Airtel (3-4 different locations with different PCIs) | Breaks model's dependence on specific PCI values and band combinations | Medium (need to travel) |
| **3** | Different network operator (Jio) | Different eNB vendor = different configuration philosophy and reconfig behavior | Medium (need Jio SIM) |
| **4** | Different device (Samsung Galaxy with Qualcomm) | Different modem firmware affects MAC-layer scheduling request behavior | Medium (need another device) |
| **5** | 5G NR captures | 5G NR has fundamentally different RRC procedures (TS 38.331 vs TS 36.331). Separate model branch or transfer learning needed | High (may need 5G coverage + NR-capable MobileInsight) |

### 10.3 Recommended Next Capture Session (Phase D)

For maximum dataset improvement with minimum effort:

```
Phase D Capture Plan:
├── Device: Same OnePlus 11R
├── Network: Same Airtel
├── Location: Same cell site
├── New log codes: 0xB179, 0xB193 (PHY measurements)
├── New parser fields: structured RRC content booleans
├── Scenarios: Same Phase C chaos script (19 mixed activities)
├── Repetitions: 5 sessions at different times of day
│   ├── Session 1: Morning (8-9 AM)
│   ├── Session 2: Midday (12-1 PM)
│   ├── Session 3: Evening peak (6-7 PM)
│   ├── Session 4: Late night (11 PM - 12 AM)
│   └── Session 5: Weekend afternoon
└── Estimated capture time: ~2.5 hours total (5 x ~30 min)
```

This single capture campaign would:
- Add PHY features for handover target prediction
- Add structured reconfig content labels for Stage 2
- Provide 5x more chaos data with time-of-day diversity
- Total positive events would grow from ~3,471 to ~7,500+

---

## 11. Deployment Considerations

### 11.1 Inference Latency Requirements

The prediction must be available **before** the eNB sends the RRC Reconfiguration:

```
eNB Decision Timeline:
T=0ms:     UE sends Measurement Report
T+5-50ms:  eNB processes report, makes reconfig decision
T+10-100ms: eNB sends RRC Reconfiguration to UE

→ Prediction must complete within 10ms at most
→ This rules out server-side inference (network RTT alone is 10-30ms on LTE)
```

### 11.2 On-Device Model Constraints

The model runs on the phone's Application Processor (AP), not the baseband (locked-down, no third-party code).

| Constraint | Requirement |
|-----------|-------------|
| Model size | <500KB |
| Inference time | <5ms on mid-range ARM CPU |
| GPU required | No (battery drain) |
| Dependencies | Minimal (ONNX Runtime or native LightGBM) |

**Both recommended models meet these constraints:**

| Model | Size | Inference (ARM) | Viable? |
|-------|------|----------------|---------|
| LightGBM (100 trees, 63 leaves) | ~200KB | <1ms | **Yes** |
| TCN (3 layers, 32 channels, float16) | ~200KB | 2-4ms | **Yes** |
| Transformer (2 layers, 4 heads) | ~500KB+ | 5-10ms | Borderline |

### 11.3 Data Access Bottleneck on Live Devices

**This is the real deployment challenge — not the model itself.**

| Data Source | MAC-Layer Access | Root Required | Production Viable |
|-------------|-----------------|---------------|-------------------|
| `/dev/ttyUSB0` (current method) | Full | Yes + USB debug | No (test only) |
| Android TelephonyManager API | No (RSRP/RSRQ/PCI only) | No | Yes, but coarser features |
| Qualcomm AI Engine (DSP) | Full (modem-internal) | Chipset vendor partnership | Yes, but requires Qualcomm |
| O-RAN near-RT RIC (xApp) | Full (network-side) | No (runs in RAN) | Yes, for operators |

### 11.4 Realistic Near-Term Deployment Paths

| Path | Description | Data Access | Best For |
|------|-------------|-------------|----------|
| **Network optimization tool** | Offline analysis of collected diagnostic logs. Flag unnecessary reconfigs, identify optimization opportunities | Full (post-hoc analysis) | Research, network planning |
| **Drive test companion** | Real-time prediction on rooted test phones with MobileInsight during field testing | Full (rooted device) | Network testing, coverage optimization |
| **O-RAN xApp** | Deploy model in near-RT RIC. Receives UE telemetry from CU/DU, predicts reconfigs network-side | Full (RAN visibility) | Operators, O-RAN deployments |
| **Consumer app (future)** | Retrain model on Android TelephonyManager features only (RSRP, RSRQ, throughput, PCI). Reduced accuracy but no root required | Partial | Consumer-facing |

---

## 12. Execution Roadmap

| Week | Task | Deliverable |
|------|------|-------------|
| **1-2** | Data pipeline: stream separation, C-RNTI filtering, 250ms windowing, label alignment | `pipeline.py` — validated by manually checking 20 windows against raw CSV |
| **3** | Feature engineering: implement 30+ features, compute correlation matrix, drop >0.95 correlated | `features.py` — feature matrix ready for training |
| **4** | LightGBM baseline with LOBO-CV and per-scenario metrics | Baseline AUC-PR score, feature importance plot |
| **5-6** | TCN on feature sequences, compare against LightGBM, build ensemble | Model comparison report, ensemble weights |
| **7** | Multi-label Stage 2 for reconfig content prediction | Content prediction model with per-type metrics |
| **8+** | Phase D data collection (PHY + different times of day) and retrain | Expanded dataset, improved model |

---

## Appendix A: Current 31 CSV Columns (Reference)

These are the raw columns extracted by the MobileInsight parser from MAC/RLC/RRC diagnostic packets. All ML features in Section 3 are derived from these columns.

> Exact column names should be verified against the actual CSV headers in the `.csv.gz` files.

## Appendix B: 3GPP Reference Specifications

| Spec | Topic | Relevance |
|------|-------|-----------|
| 3GPP TS 36.331 | RRC Protocol (LTE) | Defines RRC Reconfiguration message structure, measurement config, DRX config IEs |
| 3GPP TS 36.321 | MAC Protocol (LTE) | Defines DRX behavior, BSR procedures, scheduling request |
| 3GPP TS 36.322 | RLC Protocol (LTE) | Defines AM/UM modes for data vs voice bearers |
| 3GPP TS 36.323 | PDCP Protocol (LTE) | Defines ROHC for VoLTE |
| 3GPP TS 38.331 | RRC Protocol (5G NR) | Future reference when extending to 5G |
