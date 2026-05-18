# DRB Reconfiguration Contents Prediction — 

**Purpose:** Bring you from zero to "I can capture a log, parse it, train a model, and contribute"
**Author:** Venu (IIT Delhi)
**Date:** May 14, 2026

---

## Table of Contents

1. [What we are trying to do](#1-what-we-are-trying-to-do)
2. [End-to-end pipeline (10,000-ft view)](#2-end-to-end-pipeline-10000-ft-view)
3. [Hardware & software setup](#3-hardware--software-setup)
4. [Step-by-step: How we capture a log](#4-step-by-step-how-we-capture-a-log)
5. [Step-by-step: How we parse a log into a dataset](#5-step-by-step-how-we-parse-a-log-into-a-dataset)
6. [Step-by-step: How we train the models](#6-step-by-step-how-we-train-the-models)
7. [The dataset schema (82 columns)](#7-the-dataset-schema-82-columns)
8. [Bug fixes & feature additions in the last week](#8-bug-fixes--feature-additions-in-the-last-week)
9. [All Excel / CSV / Parquet files explained](#9-all-excel--csv--parquet-files-explained)
10. [Current results & limitations](#10-current-results--limitations)
11. [What you can pick up](#11-what-you-can-pick-up)
12. [Reproducibility — exact commands](#12-reproducibility--exact-commands)
13. [Common pitfalls & how to debug](#13-common-pitfalls--how-to-debug)

---

## 1. What we are trying to do

**One-line goal:** Given the recent radio-layer telemetry from a UE (User Equipment, i.e. the phone), can we **predict the parameter values inside the next RRC Reconfiguration message that the eNB will send**, *before* it actually arrives?

**The setting:**

- In LTE, when the network (eNB / base station) wants to add a Data Radio Bearer (DRB) for the UE, it sends an **`rrcConnectionReconfiguration`** message over the air.
- This message carries a bundle of parameters: RLC mode (AM/UM), poll-retransmit timer, PDCP discard timer, logical-channel priority, etc.
- These parameters fundamentally shape how the UE buffers, transmits, and acknowledges data.

**The hypothesis:** The UE side has signals (MAC scheduling pattern, PHY SNR, measurement reports, PUCCH activity) that *foreshadow* an incoming reconfiguration. If we can predict the parameters 500 ms ahead, the UE could pre-warm its radio stack and reduce reconfig latency.

**The research contribution:** *Predictive bearer management* driven entirely from UE-side observables, without any change to the eNB.

---

## 2. End-to-end pipeline (10,000-ft view)

```
┌────────────────┐    ┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Android phone │    │ /dev/ttyUSB0    │    │   .mi2log file   │    │  parse_drb_*.py │    │   .csv / .xlsx  │
│  (diag mode)   │───▶│   (Qualcomm     │───▶│   (binary, but   │───▶│   (windowing +  │───▶│   (82 columns,  │
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

## 3. Hardware & software setup

### Hardware

- **Phone:** Android device with a Qualcomm modem (we use a phone with Snapdragon 8 Gen 2 in DIAG mode).
- **USB cable** that supports data (not just charging).
- **Linux laptop** with this MobileInsight install at `/home/venu/Downloads/MobileInsight-6.0.0/`.

### Software stack

- **MobileInsight 6.0.0** (locally patched in `/home/venu/Downloads/MobileInsight-6.0.0/`). Provides:
  - `dm_collector_c` — the C++ engine that talks to the Qualcomm DIAG interface
  - `OfflineReplayer` — decodes saved `.mi2log` files into structured Python events
  - `Analyzer` — base class we subclass to receive decoded events
- **Python 3.10+** with `numpy`, `pandas`, `lightgbm`, `scikit-learn`, `openpyxl`, `pyarrow` (for parquet).

### Phone-side prerequisite

The phone must be in **DIAG mode** (sometimes called "diag-enable" or "fastboot diag" depending on model). Typical procedure:
- Root the phone (or use a developer-mode-enabled firmware that exposes DIAG)
- Enable USB diagnostic mode via the firmware-specific dialer code (e.g. `*#*#717717#*#*` on some devices) or via the device's developer settings
- Verify `/dev/ttyUSB0` shows up after `dmesg | grep ttyUSB`

If `/dev/ttyUSB0` is owned by root and you get permission denied:
```bash
sudo chmod a+rw /dev/ttyUSB0
```

---

## 4. Step-by-step: How we capture a log

### 4.1 The capture script: `dm_capture.py`

Location: `/home/venu/Downloads/MobileInsight-6.0.0/scripts/dm_capture.py`

What it does:
1. Opens the serial port `/dev/ttyUSB0` at 115200 baud.
2. Disables any existing log filters on the phone via `dm_collector_c.disable_logs(phy_ser)`.
3. Enables a whitelist of **23 Qualcomm log types** (e.g. `LTE_RRC_OTA_Packet`, `LTE_MAC_UL_Transport_Block`, `LTE_PHY_PUCCH_Tx_Report`, etc.) via `dm_collector_c.enable_logs(phy_ser, LOG_TYPES)`.
4. Sets up a filtered export — only those 23 log types get written to `session.mi2log`.
5. Streams data from the phone for `duration` seconds and writes to file.

### 4.2 The 23 log types enabled (and why each matters)

| Log type | What it carries | Why we need it |
|----------|----------------|----------------|
| `LTE_RRC_OTA_Packet` | All RRC messages (Reconfiguration, MeasurementReport, ConnectionSetup, etc.) | **THE most important log** — contains the reconfig XML we're predicting |
| `LTE_RRC_Serv_Cell_Info` | Cell ID, PCI, EARFCN | Cell context |
| `LTE_RRC_MIB_Packet` | Master Info Block | System info |
| `LTE_NAS_ESM_OTA_*` | EPS Session Management (bearer establish, modify, deactivate) | Tells us when bearers are added/removed at the upper layer |
| `LTE_NAS_EMM_OTA_*` | EPS Mobility Management (attach, TAU) | Mobility events |
| `LTE_MAC_UL_Transport_Block` | Per-subframe UL grant size, BSR, PHR, padding, LCIDs | The richest feature source — UE scheduling pattern |
| `LTE_MAC_DL_Transport_Block` | Per-subframe DL grant size | DL activity |
| `LTE_PDCP_DL_Ctrl_PDU`, `LTE_PDCP_UL_Ctrl_PDU` | PDCP control PDUs | Status reports |
| `LTE_RLC_DL_AM_All_PDU`, `LTE_RLC_UL_AM_All_PDU` | RLC AM data PDUs | Retx pattern (when decodable) |
| `LTE_RLC_UL_Config_Log_Packet` (0xB091), `LTE_RLC_DL_Config_Log_Packet` (0xB081) | Current bearer config snapshot | **Originally for `cur_rlc_*` features — currently unusable due to Snapdragon 8 Gen 2 format change** |
| `LTE_RLC_UL_Stats` (0xB097), `LTE_RLC_DL_Stats` (0xB087) | Aggregate retx counts | Same — format incompatible |
| `LTE_PHY_PUCCH_Tx_Report` (0xB13C) | Per-subframe PUCCH transmissions: SR, ACK, NACK, Format, Tx power | **Live and working** — SR count and Tx power are real features |
| `LTE_PHY_PUCCH_Power_Control` (0xB16F) | PUCCH power control events | Power features |
| `LTE_PHY_PUCCH_CSF` (0xB14D) | CQI feedback over PUCCH | Channel quality |
| `LTE_PHY_Serv_Cell_Measurement` | RSRP / RSRQ / SNR | PHY-quality features |
| `LTE_PHY_PUSCH_CSF` | Wideband CQI | Channel quality |
| `LTE_PHY_Connected_Mode_Intra_Freq_Meas` | Intra-frequency neighbour measurements | Handover precursors |

### 4.3 Capturing a session — actual commands

```bash
# Step 1: Plug phone in DIAG mode, verify it shows up
ls /dev/ttyUSB*
# expected: /dev/ttyUSB0

# Step 2: Fix permissions (if owned by root)
echo "<your-password>" | sudo -S chmod a+rw /dev/ttyUSB0

# Step 3: Create an output directory
mkdir -p /home/venu/Desktop/ueautomation/logs/MyCapture_$(date +%Y%m%d_%H%M%S)
cd /home/venu/Desktop/ueautomation/logs/MyCapture_*

# Step 4: Run the capture for 180 seconds
python3 /home/venu/Downloads/MobileInsight-6.0.0/scripts/dm_capture.py \
    /dev/ttyUSB0 115200 180 ./session.mi2log

# Step 5 (CRITICAL): While the capture is running, USE THE PHONE
# - Start a VoLTE call, drop it, restart it (forces DRB add/remove)
# - Open a browser, scroll, close (data DRB usage)
# - Toggle airplane mode (forces re-attach and DRB re-setup)
# - Move to a weak-signal location and back (forces handover → reconfig)
#
# The model is bad at predicting parameters when very few DRB events occur.
# Aim for 20+ DRB setup events per capture.

# Step 6: Optionally save activity labels to labels.csv (helps the model
# know "this window was during a phone call vs idle")
# Format: Timestamp_Epoch, State, Activity
# e.g.    1715600000.0, START, VoLTE
#         1715600060.0, STOP,  VoLTE
```

### 4.4 What success looks like

After capture you should see a `.mi2log` file in the 1–100 MB range. The bigger captures (Phase_D_Batch_2 was 97 MB) yield more events. The smaller ones (Phase_E_Batch_2 is 1.7 MB) yield fewer DRB events.

A quick sanity check:
```bash
ls -lh ./session.mi2log
# If the file is 0 bytes → the phone's DIAG isn't streaming. Check ttyUSB0.
# If < 100 KB → maybe capture aborted early. Check the .capture.log.
```

---

## 5. Step-by-step: How we parse a log into a dataset

### 5.1 The parser: `parse_drb_contents.py`

Location: `/home/venu/Desktop/ueautomation/scripts/parse_drb_contents.py`

Internally it does 4 things:

1. **Replay the .mi2log file** via MobileInsight's `OfflineReplayer`. This streams decoded events one at a time into our `RawEventCollector` subclass.
2. **Bucket events into typed lists** — `mac_events`, `phy_serv_events`, `phy_cqi_events`, `rrc_events`, `nas_events`, `pucch_events`, plus state-tracking lists `rrc_drb_state_events` and `rrc_reconf_type_events`.
3. **Generate sliding windows** — every 50 ms, take a 500 ms-wide window ending at this point, aggregate all the events that fall inside it into features (sums, means, counts, deltas).
4. **Look forward 500 ms** for each window to find the next RRC Reconfiguration. If one exists, extract its parameters from the XML into the 25 output columns. If not, all outputs are zero / "none".

### 5.2 Key concepts to understand

**Sliding window:**
- Window width: `WINDOW_MS = 500`
- Stride: `STRIDE_MS = 50`
- Horizon: `HORIZON_MS = 500` (how far into the future we look for the label)

So at time *t*, the input is everything that happened in `[t-500ms, t]`, and the label is "what reconfig (if any) happens in `[t, t+500ms]`."

**Why 500 ms:** Long enough to aggregate multiple MAC TBs and meas reports, short enough that the model isn't predicting too far into the future.

**Why 50 ms stride:** Gives ~20 windows per second of capture. Adjacent windows overlap heavily, so you get ~10× more training examples than non-overlapping windows.

### 5.3 Output extraction (the 25 output columns)

When a window's horizon contains an RRC Reconfiguration, the parser calls `extract_drb_params(xml_root)` which walks the Wireshark-style XML tree and pulls out:

- `out_has_reconf` — 1 (always, since we found one)
- `out_reconf_ts` — exact timestamp of the reconfig
- `out_num_drbs_added` — count of `drb-Identity` fields
- `out_has_drb_release` — was there a `drb-ToReleaseList`?
- `out_drb_id`, `out_eps_bearer_id` — identity of the first DRB added
- `out_rlc_mode` — "AM" / "UM" / "TM" / "none" (parsed from the XML tag names)
- `out_rlc_t_poll_retx_ms`, `out_rlc_poll_pdu`, etc. — RLC AM-specific params
- `out_pdcp_discard_timer_ms`, `out_pdcp_status_report`, `out_pdcp_rohc_enabled`
- `out_lc_priority`, `out_lc_pbr_kbps`, `out_lc_bsd_ms`, `out_lc_group`
- `out_has_meas_config`, `out_has_phy_config`, `out_has_mac_config`, `out_has_handover`

If no reconfig in the next 500 ms → all numeric outputs are 0, string outputs are "none".

### 5.4 Running the parser

```bash
python3 /home/venu/Desktop/ueautomation/scripts/parse_drb_contents.py \
    /home/venu/Desktop/ueautomation/logs/MyCapture_X/session.mi2log \
    /home/venu/Desktop/ueautomation/outputs/batches/batch_11_mycapture.csv \
    --batch-id 11
```

What you should see in the output:
```
[1/3] Parsing .mi2log ...
  MAC events:         12,345
  RRC messages:       234
  RRC reconfigurations: 45
  DRB setup events:   12
  PHY serv-cell meas: 3,456
  ...
[2/3] Building windows ...
  Generating 3,500 windows (stride 50ms)...
[3/3] Writing output ...

  Total windows:           3,500
  Windows with reconf:     230 (6.57%)
  Windows with DRB setup:  20
```

> **Healthy capture indicators:** MAC events > 5,000; RRC reconfigurations > 10; DRB setup events > 5. If MAC events == 0, the capture file is corrupt (this happened to Phase_C_Batch_1_Chaos_20260508 — `dm_capture.py` crashed before writing real data).

---

## 6. Step-by-step: How we train the models

### 6.1 The trainer: `train_drb_contents.py`

Location: `/home/venu/Desktop/ueautomation/scripts/train_drb_contents.py`

It does:
1. Loads `drb_contents_combined.parquet` (all batches merged).
2. One-hot encodes the `current_activity` column (Browsing / VoiceCall / DataToggle / Idle / etc.).
3. Splits by batch_id: **train = batches 5,6,7,8,9,10 | val = batch 1 | test = batches 2,4**. We use batch-level splits (not random row splits) to avoid leakage — adjacent overlapping windows would otherwise contaminate val/test.
4. For each of the 10 output targets, calls `_train_one()`:
   - For **binary** targets: trains with `objective="binary"`, `scale_pos_weight = n_neg / n_pos` to handle imbalance, optimizes AUC-PR via `average_precision`.
   - For **multiclass** targets: trains with `objective="multiclass"`, with **inverse-frequency sample weighting** so rare classes aren't ignored.
   - Early stopping with patience=40 on the val set.
5. Evaluates each model on val and test, prints AUC-PR / accuracy / confusion matrix.
6. Saves: `drb_contents_models.pkl` (full model bundle), one `model_<target>.txt` per output (human-readable), and a feature-importance CSV.

### 6.2 The 10 output targets we train

```python
OUTPUT_TARGETS = [
    ("out_has_reconf",            "binary",     None),                       # primary
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

### 6.3 Why one model per target (not a single multi-output model)

LightGBM doesn't natively support multi-output. Wrapping it would also force every target to share the same hyperparameters, which doesn't match reality — e.g., RLC mode is 3-class with heavy imbalance, but `out_has_reconf` is binary. One-model-per-target is the simplest correct approach and lets us see per-target metrics.

### 6.4 Running it

```bash
cd /home/venu/Desktop/ueautomation
python3 scripts/train_drb_contents.py
```

Output is interactive — each target prints its training progress, val/test metrics, then a final top-15 feature importance table.

---

## 7. The dataset schema (82 columns)

Every row in the dataset represents one 500 ms window, with **57 input features** and **25 output targets**.

### 7.1 Input features (what the model sees)

| Group | Columns |
|-------|---------|
| **Meta**       | `window_id`, `batch_id`, `window_ts` |
| **MAC**        | `mac_event_count`, `total_grant_bytes`, `mean_grant_bytes`, `max_grant_bytes`, `grant_variance`, `sbsr_count`, `lbsr_count`, `phr_count`, `padding_ratio`, `unique_harq_ids`, `num_data_lcids_max`, `grant_bytes_delta`, `bsr_rate_change` |
| **RLC** (placeholder, currently zero) | `rlc_total_pdu`, `rlc_data_pdu`, `rlc_retx_pdu`, `rlc_poll_count`, `rlc_nack_count`, `rlc_ctrl_pdu` |
| **Current RLC config state** (from RRC XML tracking, now working) | `cur_rlc_mode`, `cur_t_poll_retx_ms`, `cur_poll_pdu`, `cur_poll_byte`, `cur_max_retx_threshold`, `cur_t_reordering_ms`, `cur_t_status_prohibit_ms`, `cur_sn_length`, `cur_num_active_drbs` |
| **PUCCH**      | `pucch_sr_count`, `pucch_ack_count`, `pucch_nack_count`, `pucch_mean_tx_power` |
| **PHY**        | `phy_rsrp_mean`, `phy_rsrq_mean`, `phy_snr_rx0_mean`, `phy_cqi_mean`, `phy_rsrp_delta` |
| **RRC**        | `meas_report_count`, `reconf_in_window`, `time_since_last_reconf_ms` |
| **Temporal (NEW)** | `reconf_type_hist_0`, `reconf_type_hist_1`, `reconf_type_hist_2`, `meas_report_acceleration`, `inter_reconf_slope_ms`, `drb_3_age_ms`, `drb_4_age_ms`, `drb_5_age_ms`, `drb_6_age_ms`, `drb_7_age_ms` |
| **NAS**        | `nas_esm_in_window` |
| **DRX**        | `time_since_last_mac_ms` |
| **Activity**   | `current_activity` (string), `time_into_activity_ms` |

### 7.2 Output targets (the labels)

| Group | Columns |
|-------|---------|
| **Label flag** | `out_has_reconf`, `out_reconf_ts`, `out_num_drbs_added`, `out_has_drb_release` |
| **DRB identity** | `out_drb_id`, `out_eps_bearer_id` |
| **RLC config** | `out_rlc_mode`, `out_rlc_t_poll_retx_ms`, `out_rlc_poll_pdu`, `out_rlc_poll_byte`, `out_rlc_max_retx`, `out_rlc_t_reordering_ms`, `out_rlc_t_status_prohibit_ms`, `out_rlc_sn_bits` |
| **PDCP config** | `out_pdcp_discard_timer_ms`, `out_pdcp_status_report`, `out_pdcp_rohc_enabled` |
| **LC config** | `out_lc_priority`, `out_lc_pbr_kbps`, `out_lc_bsd_ms`, `out_lc_group` |
| **Reconf flags** | `out_has_meas_config`, `out_has_phy_config`, `out_has_mac_config`, `out_has_handover` |

---

## 8. Bug fixes & feature additions in the last week

### 8.1 Bug: `cur_rlc_*` state tracking was broken (CRITICAL)

**Symptom:** All 9 columns of `cur_rlc_*` were zero across the entire dataset.

**Root cause:** In `parse_drb_contents.py`, the `_handle_rrc` method was appending a state snapshot to `rrc_drb_state_events` **on every reconfiguration**, including measurement-only reconfigs. Measurement-only reconfigs don't carry DRB params, so `drb_params["out_rlc_mode"]` would be `"none"` and `cur_state["rlc_mode"]` would be `0`. This overwrote the valid AM/UM state from the previous true DRB-setup reconfig.

**Fix:**
```python
if rlc_mode_str in ("AM", "UM"):     # ← only update state when DRB config actually changed
    cur_state = { ... }
    self.rrc_drb_state_events.append((ts_s, cur_state))
```

**Evidence it worked:** Before fix: 1/6262 windows had `cur_rlc_mode != 0`. After fix: 6225/6262 (95.6%).

**Is this correct?** Yes. Measurement reconfigs and PHY-config-only reconfigs do not change the RLC bearer state per 3GPP TS 36.331, so they should not reset our state tracker. The fix matches the standard's semantics.

---

### 8.2 Bug: `cur_poll_byte` was always zero

**Symptom:** Even after fix #1, `cur_poll_byte` stayed at 0.

**Root cause:** Hardcoded `"poll_byte": 0` inside the `cur_state` dict — we forgot to convert `out_rlc_poll_byte` (string like `"kBinfinity"`) to its integer kB value.

**Fix:** Added helper function:
```python
def _poll_byte_kb(s: str) -> int:
    """pollByte enum string → integer kB value. kBinfinity → 100000."""
    if not s or s == "0": return 0
    if 'infinity' in str(s).lower(): return 100000
    m = re.search(r'kB(\d+)', str(s), re.IGNORECASE)
    return int(m.group(1)) if m else 0
```

Then in `cur_state`: `"poll_byte": _poll_byte_kb(drb_params.get("out_rlc_poll_byte", "0"))`.

**Is this correct?** Yes, this matches the LTE-RRC ASN.1 enum definition. `kB25` → 25 kB threshold; `kBinfinity` → effectively no threshold (we encode as 100000).

---

### 8.3 Bug: `_join_activity` crashed when `labels.csv` was empty

**Symptom:** Parser crashed during the activity-join step when a capture had no `labels.csv` activities.

**Fix:** Added an early-return guard:
```python
if not activities or len(df) == 0:
    return df
```

**Is this correct?** Yes — defensive guard for an edge case that occurs in unlabeled captures.

---

### 8.4 Bug: Mixed types in combined parquet for `out_rlc_poll_pdu`, `out_rlc_poll_byte`, `out_rlc_max_retx`

**Symptom:** After concatenating batch CSVs, pandas complained about mixed dtypes. Some rows had `0` (int) for the "no reconf" case, others had strings like `"p64"` for actual reconfigs.

**Fix:** Force string conversion in the combiner:
```python
for c in ["out_rlc_poll_pdu", "out_rlc_poll_byte", "out_rlc_max_retx", "out_rlc_mode"]:
    df[c] = df[c].astype(str)
```

**Is this correct?** Yes — these are categorical/enum outputs, so string is the right type.

---

### 8.5 Feature addition: 10 new temporal / rate-of-change features

These were added because the original feature set was all "instantaneous snapshots" of state. The professor's original research notes had identified 4 categories of *trajectory* features that we hadn't implemented:

| Feature | Why it should help |
|---------|-------------------|
| `reconf_type_hist_0/1/2` | "Recent burst of measurement reconfigs" is a strong precursor to handover. By exposing the last 3 reconf types, the model can spot escalation patterns. |
| `meas_report_acceleration` | 2nd derivative of meas report rate. UEs that are about to hand over emit a burst of measurement reports — the *acceleration* of reports is what matters, not just the count. |
| `inter_reconf_slope_ms` | When reconfigs cluster (gap shrinking), a DRB add or HO is often imminent. Linear-fit slope on the last 3 gaps captures this. |
| `drb_3..7_age_ms` | Session context — a freshly-added DRB behaves differently from one that's been active for 60 seconds. |

**Are these features correct?** Yes, with one caveat. They are theoretically grounded (each has a clear LTE behavioral basis), and the empirical evidence is strong: **3 of the 10 new features ranked in the top 5 by gain in the trained model immediately**, displacing all but one of the original RRC features.

Caveat: `time_since_last_reconf_ms` still dominates the feature importance at 47K gain — there's a risk that the model is mostly learning the inter-event distribution rather than radio-state → reconfig content. This needs a leakage audit.

---

## 9. All Excel / CSV / Parquet files explained

Under `/home/venu/Desktop/ueautomation/outputs/`:

### Per-batch raw CSVs (`outputs/batches/`)

| File | Rows | Source mi2log |
|------|------|---------------|
| `batch_1_phase_d1.csv` | 6,262 | Phase_D_Batch_1 (24 MB capture) |
| `batch_2_phase_d2.csv` | 18,241 | Phase_D_Batch_2 (97 MB capture) — largest batch |
| `batch_4_download.csv` | 3,283 | Scenario: Download (4.8 MB) |
| `batch_5_browsing.csv` | 1,345 | Scenario: Browsing (3.2 MB) |
| `batch_6_voicecall.csv` | 755 | Scenario: VoiceCall (261 KB) |
| `batch_7_datatoggle.csv` | 820 | Scenario: DataToggle (73 KB) |
| `batch_8_verify.csv` | 1,156 | verify_test/session.mi2log (replaces corrupted Phase_C_Chaos) |
| `batch_9_pucch_test.csv` | 3,561 | Phase_E_Quick (PUCCH test capture) |
| `batch_10_phase_e2.csv` | 4,493 | Phase_E_Batch_2 (NEW this week) |

> Each CSV is 82 columns. Same schema across all batches. `batch_3` does not exist — that was an originally-empty Phase_C capture, skipped.

### Combined dataset

| File | Rows | Description |
|------|------|-------------|
| `drb_contents_combined.parquet` | 39,916 | All 9 batches concatenated. **This is what the trainer loads.** Parquet is ~10× smaller than CSV, fast to load. |

### Inspection xlsx files (for showing the dataset to humans)

| File | Rows | Purpose |
|------|------|---------|
| `drb_features_phase_d1_82col.xlsx` | 6,262 | Full Phase_D Batch 1, every window (reconf + non-reconf). For looking at one complete batch in Excel. |
| `drb_features_reconf_only.xlsx` | 6,867 | All windows across 9 batches where `out_has_reconf = 1`. Includes meas-only and PHY-config reconfigs (DRB output cols are blank for those). |
| `drb_features_drb_setup_only.xlsx` | 331 | Only the windows where a real DRB setup is in the label. **This is the "ground truth" subset — every row has meaningful output values.** RLC mode distribution: AM=228, UM=61, none=42. → This is the one we send to the professor. |

### Model artifacts

| File | What it is |
|------|-----------|
| `drb_contents_models.pkl` | Pickled dict `{"models": ..., "encoders": ..., "features": ...}` — load this to make predictions |
| `model_out_*.txt` | One per output target, LightGBM's own text dump (human-readable tree structure) |
| `drb_contents_feature_importance.csv` | Gain-based feature ranking from `out_has_reconf` model |

### Summary documents (created this week)

| File | Purpose |
|------|---------|
| `drb_weekly_summary_may14.md` | Week-in-review report for the professor (markdown) |
| `drb_weekly_summary_may14.xlsx` | Same, but as a 9-sheet Excel workbook |
| `excel_files_guide.md` | Notes on which xlsx to send to the professor |
| `onboarding_phd_student.md` | ← This file |

---

## 10. Current results & limitations

### 10.1 Results table

| Target | Val | Test | Interpretation |
|--------|------|------|----------------|
| `out_has_reconf` (binary)            | AUC-PR **0.299** | AUC-PR **0.231** | Above random (which would be ~0.17). Modest signal. |
| `out_has_meas_config` (binary)       | AUC-PR 0.398     | AUC-PR 0.242     | Slightly better than `has_reconf`. |
| `out_rlc_mode` (3-class) on DRB win  | 20.9%            | 32.8%            | At or below random baseline (33%) — model is not learning meaningful AM-vs-UM signal. |
| `out_lc_priority` on DRB win         | 26.9%            | 24.6%            | Same — data-limited. |
| Other DRB-param multiclass targets   | 22–34%           | 24–35%           | All hover around random. |

### 10.2 Top 5 features by gain (out_has_reconf model)

1. `time_since_last_reconf_ms` — 47,204
2. `time_since_last_mac_ms` — 23,838
3. **`drb_3_age_ms` (NEW)** — 18,851
4. **`inter_reconf_slope_ms` (NEW)** — 18,781
5. **`drb_4_age_ms` (NEW)** — 16,665

### 10.3 Honest limitations

- **331 DRB events is too few** for 25 output targets with 3-class classification. Random baseline for 3-class is 33%, and we're hovering there.
- **eNB uses templated configs.** Most DRB setups carry identical canonical parameter sets — most of the predictability is just "memorize the carrier's defaults," which doesn't need ML.
- **PUCCH ACK/NACK are zero** across all 9 captures — no Format 1A/1B traffic appears in our captures. Either the feature decoding is incomplete or the captures don't exercise UL HARQ enough.
- **`rlc_*_pdu` placeholders** — Snapdragon 8 Gen 2 RLC Config binary format is undecodable. We bypass this by tracking state through RRC XML (which works), but lose dynamics (retx counts, NACKs).

---

## 11. What you can pick up

Concrete tasks ranked by leverage. The first three are doable with **just the existing dataset** — no new captures needed.

### Tier 1 — Use existing data, high impact

1. **Add a reconfig-type classifier as a new output target** (4-class: meas / DRB-add / PHY-config / HO). You'd have 6,867 positives instead of 331. The new temporal features were designed for this. Result should be clearly above random. Expected work: ~1 day.

2. **Audit `time_since_last_reconf_ms` for leakage.** It dominates feature importance by a huge margin. Question: is it truly an input feature, or is the model just learning the inter-event interval distribution and ignoring radio state? Try training a model *without* this feature and see what happens to AUC-PR.

3. **Two-stage hierarchical model.** Stage 1: predict `out_has_reconf`. Stage 2: conditional on positives, predict DRB params using a separate model trained only on DRB-setup windows (331 of them). This concentrates learning capacity where it matters. Expected work: 2–3 days.

### Tier 2 — Better data

4. **Run more targeted captures.** We need >1000 DRB events to push past random baseline on multiclass. Script:
   - Repeated VoLTE call setup/teardown (each call adds DRB 2)
   - Repeated mobile-data toggle off-on (each cycle adds DRB 3)
   - Forced handover via signal-attenuating (Faraday bag, walk away from window)
   - Aim: 4–6 captures of 3 minutes each, 50+ DRB events per capture.

5. **Add Format 1A/1B PUCCH decoding.** Right now `pucch_ack_count` and `pucch_nack_count` are zero — either the decoder isn't pulling the right field or the captures don't exercise UL HARQ. Investigate.

### Tier 3 — Deep dives

6. **Reverse-engineer the Snapdragon 8 Gen 2 RLC Config inner-subpacket format.** This is the blocker for `rlc_total_pdu`, `rlc_retx_pdu`, etc. Hard problem (proprietary binary format), but if cracked it would 10× the feature richness.

7. **Replace LightGBM with a sequence model.** Currently each window is treated independently. A small TCN or GRU over the last 5 windows might capture temporal context the feature-engineered approach misses.

---

## 12. Reproducibility — exact commands

```bash
# ── 0. One-time setup ──────────────────────────────────────────
# (Already done on Venu's machine. Listed here for reference.)
pip install numpy pandas lightgbm scikit-learn openpyxl pyarrow

# ── 1. Capture a new log ──────────────────────────────────────
mkdir -p /home/venu/Desktop/ueautomation/logs/MyCapture_$(date +%Y%m%d_%H%M%S)
cd /home/venu/Desktop/ueautomation/logs/MyCapture_*
echo "<sudo-password>" | sudo -S chmod a+rw /dev/ttyUSB0
python3 /home/venu/Downloads/MobileInsight-6.0.0/scripts/dm_capture.py \
    /dev/ttyUSB0 115200 180 ./session.mi2log

# ── 2. Parse it ───────────────────────────────────────────────
python3 /home/venu/Desktop/ueautomation/scripts/parse_drb_contents.py \
    ./session.mi2log \
    /home/venu/Desktop/ueautomation/outputs/batches/batch_11_mycapture.csv \
    --batch-id 11

# ── 3. Re-parse all existing batches (only needed after parser changes)
bash /home/venu/Desktop/ueautomation/scripts/reparse_all.sh

# ── 4. Rebuild the combined parquet ───────────────────────────
python3 - << 'EOF'
import pandas as pd, os
DIR = "/home/venu/Desktop/ueautomation/outputs/batches"
files = sorted([f for f in os.listdir(DIR) if f.endswith(".csv")])
dfs = [pd.read_csv(os.path.join(DIR, f), low_memory=False) for f in files]
for df in dfs:
    for c in ["out_rlc_poll_pdu","out_rlc_poll_byte","out_rlc_max_retx","out_rlc_mode"]:
        df[c] = df[c].astype(str)
combined = pd.concat(dfs, ignore_index=True)
combined["window_id"] = range(len(combined))
combined.to_parquet("/home/venu/Desktop/ueautomation/outputs/drb_contents_combined.parquet", index=False)
print(f"Combined: {len(combined):,} × {len(combined.columns)}")
EOF

# ── 5. Train all 10 models ────────────────────────────────────
cd /home/venu/Desktop/ueautomation
python3 scripts/train_drb_contents.py

# ── 6. Inspect results ────────────────────────────────────────
# Top features:
cat /home/venu/Desktop/ueautomation/outputs/drb_contents_feature_importance.csv | head -20

# Load a trained model for prediction:
python3 - << 'EOF'
import pickle
with open("/home/venu/Desktop/ueautomation/outputs/drb_contents_models.pkl","rb") as f:
    bundle = pickle.load(f)
print("Targets:", list(bundle["models"].keys()))
print("Features:", len(bundle["features"]))
EOF
```

---

## 13. Common pitfalls & how to debug

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Phone not in DIAG mode | `dm_capture.py` exits immediately or no `ttyUSB0` shows up | Check phone firmware; some need a dialer code or developer toggle |
| `/dev/ttyUSB0` permission denied | "Permission denied" when running capture | `sudo chmod a+rw /dev/ttyUSB0` |
| Capture crashes mid-recording | "Wrong type name" in the capture log; .mi2log file present but parser shows 0 events | Some log types aren't valid for that firmware. Check the `LOG_TYPES` list in `dm_capture.py`; comment out the failing one. |
| No DRB events in capture | Parser shows "DRB setup events: 0" | The phone didn't exercise bearer changes during the capture window. Trigger explicit events (VoLTE call, data toggle, airplane mode). |
| All `cur_rlc_*` features are 0 | Re-parsed dataset still has zeros | Make sure you're using the latest `parse_drb_contents.py` (check for the `if rlc_mode_str in ("AM", "UM"):` guard) |
| Training AUC-PR drops with more data | "I added a batch and the model got worse" | Check the new batch's reconf rate. If it's very different from val/test, the model is now adapting to a different distribution. |
| `pyarrow` parquet errors | "could not infer schema" | Run the type-fix snippet in step 4 of the reproducibility commands |

---

## 14. Quick mental model: what does the model actually do?

The model is trying to learn:

> "Given the radio-layer activity of the UE in the last 500 ms (MAC scheduling pattern, PHY SNR, recent reconfigs, PUCCH activity, current bearer state, session age, etc.), is the eNB about to send a Reconfiguration in the next 500 ms? And if so, what parameters will it carry?"

In LTE terms: it's modeling the eNB's reconfiguration policy purely from UE-observable side-channels.

**Why this is plausible:** The eNB doesn't fire reconfigurations randomly. They follow measurement reports, congestion patterns, and bearer-add requests from the EPC. All of those have UE-side fingerprints.

**Why this is hard:** The exact decision logic lives inside proprietary eNB firmware (Nokia, Ericsson, Huawei differ). The UE only sees the *output* of that policy. So we're trying to reverse-engineer a vendor-specific scheduling policy from indirect observations, with very few examples (331 DRB setups).

---

# DRB Reconfiguration Contents Prediction — Weekly Summary

**Period:** May 8 – May 14, 2026
**Researcher:** Venu (IIT Delhi)

---

## 1. Overview

**Goal:** Predict the parameter values inside an RRC Reconfiguration carrying a DRB Setup, 500 ms before the eNB sends it, using only UE-side observables (MAC / PHY / RRC / NAS / PUCCH).

**Pipeline:** `.mi2log` → `parse_drb_contents.py` → 82-col CSV → `train_drb_contents.py` → 10 LightGBM models (one per output target).

**Status:** Pipeline complete and reproducible. Binary "reconf-will-occur" sub-task partially working. Full 25-output content prediction is **data-limited** (only 331 DRB-setup events across 9 captures).

---

## 2. Work Completed

| Category | Item | Detail |
|----------|------|--------|
| Parser     | Built `parse_drb_contents.py` end-to-end | Reads `.mi2log`; extracts MAC / PHY / RRC / NAS / PUCCH features in 500 ms / 50 ms sliding windows; parses 25 DRB output params from RRC XML. |
| Bug fix    | `cur_rlc_*` state tracking               | `_handle_rrc` was appending state on every reconf (including meas-only), overwriting valid AM/UM with `rlc_mode=0`. Added "AM/UM only" guard. Result: **1/6262 → 6225/6262 valid**. |
| Bug fix    | `cur_poll_byte` always 0                 | Hardcoded in `cur_state` dict. Added `_poll_byte_kb()` helper, wired in. Now populated correctly. |
| Bug fix    | `_join_activity` empty-df crash          | Added early-return guard `if not activities or len(df)==0`. |
| Bug fix    | Mixed-type `out_rlc_poll_pdu/byte/max_retx` | CSV round-trip mixed `int 0` with strings like `"p64"`. Fixed in combiner via `.astype(str)`. |
| Trainer    | Built `train_drb_contents.py`            | Multi-output LightGBM: one model per target. Inverse-frequency class weighting (multiclass), `scale_pos_weight` (binary), early stopping on val. |
| Features   | Added 10 new temporal / rate-of-change features | `reconf_type_hist_0/1/2`, `meas_report_acceleration`, `inter_reconf_slope_ms`, `drb_3..7_age_ms` — captures trajectory / recency, not just snapshot state. |
| Dataset    | Re-parsed all captures, combined         | 9 batches → `drb_contents_combined.parquet` (39,916 windows × 82 cols). |
| New capture | Phase_E_Batch_2 (May 13, 1.7 MB) added as `batch_10` | 4,493 windows, 7 DRB-setup events; included in training set. |
| Training   | Multi-output LightGBM trained + evaluated | Train: batches 5–10 (12,130 win) \| Val: batch 1 (6,262) \| Test: batches 2,4 (21,524). Results in section 5. |
| Output     | Generated 3 inspection xlsx files        | `drb_features_phase_d1_82col.xlsx`, `drb_features_reconf_only.xlsx`, `drb_features_drb_setup_only.xlsx` (the 331-row "ground truth" subset). |

---

## 3. Dataset — Before vs After

| Metric                  | Before (May 8)                  | After (May 14)           |
|-------------------------|---------------------------------|--------------------------|
| Total batches           | 8                               | **9** (added Phase_E_Batch_2) |
| Total windows           | 35,768                          | **39,916**               |
| Columns                 | 72                              | **82** (+10 temporal features) |
| Reconf windows          | 6,771                           | **6,867**                |
| DRB-setup windows       | 283                             | **331**                  |
| RLC modes (DRB only)    | n/a                             | AM = 228, UM = 61, none = 42 |
| `cur_rlc_mode` non-zero | ~0%                             | **95.6%**                |
| Activity labels         | Yes (where `labels.csv` present) | Same                    |

---

## 4. New Features Added

| Feature | Definition | What it captures |
|---------|-----------|------------------|
| `reconf_type_hist_0` | Most recent reconf type before window (0=none, 1=meas, 2=DRB setup, 3=PHY/MAC, 4=HO) | Captures sequential context — was the trend toward measurement or bearer changes? |
| `reconf_type_hist_1` | 2nd most recent reconf type | (same) |
| `reconf_type_hist_2` | 3rd most recent reconf type | (same) |
| `meas_report_acceleration` | 2nd finite difference of `meas_report_count` across windows | Detects accelerating measurement reports — often precedes handover or DRB reconf. |
| `inter_reconf_slope_ms` | Slope of linear fit on last 3 inter-reconf gaps (ms/gap) | Negative → gaps shrinking → burst incoming. Positive → settling down. |
| `drb_3_age_ms` | ms since DRB-3 first activated (0 if never seen) | Session context — long-lived DRBs behave differently from newly added ones. |
| `drb_4_age_ms` | (same for DRB-4) | (same) |
| `drb_5_age_ms` | (same for DRB-5) | (same) |
| `drb_6_age_ms` | (same for DRB-6) | (same) |
| `drb_7_age_ms` | (same for DRB-7) | (same) |

---

## 5. Training Results

| Target | Task | Val | Test | Meaning |
|--------|------|------|------|---------|
| `out_has_reconf`            | binary     | **AUC-PR 0.299** | **AUC-PR 0.231** | Will any reconfig occur in next 500 ms |
| `out_has_meas_config`       | binary     | AUC-PR 0.398     | AUC-PR 0.242     | Will reconfig contain `measConfig` |
| `out_has_mac_config`        | binary     | AUC-PR 0.105     | AUC-PR 0.058     | Will reconfig contain MAC config |
| `out_rlc_t_poll_retx_ms`    | binary     | AUC-PR 0.301     | AUC-PR 0.048     | Will `t-PollRetransmit` be set |
| `out_pdcp_rohc_enabled`     | binary     | AUC-PR 0.002     | AUC-PR 0.002     | Will RoHC be enabled (very rare) |
| `out_rlc_mode`              | 3-class    | 20.9% on DRB win | 32.8% on DRB win | AM / UM / none |
| `out_rlc_t_reordering_ms`   | multiclass | 20.9% on DRB win | 32.8% on DRB win | (correlated with `rlc_mode`) |
| `out_lc_priority`           | multiclass | 26.9% on DRB win | 24.6% on DRB win | 1–16 priority |
| `out_lc_group`              | multiclass | 25.4% on DRB win | 34.4% on DRB win | Logical channel group |
| `out_pdcp_discard_timer_ms` | multiclass | 26.9% on DRB win | 29.5% on DRB win | PDCP discardTimer |

> Random baseline for the 3-class multiclass tasks is ~33%. Current accuracy hovers near baseline → the parameter-prediction models are not learning meaningful signal yet, primarily because of low DRB-event volume.

---

## 6. Top Features by Gain (`out_has_reconf` model)

| Rank | Feature | Gain |
|------|---------|------|
| 1  | `time_since_last_reconf_ms`      | 47,204.3 |
| 2  | `time_since_last_mac_ms`         | 23,837.8 |
| 3  | **`drb_3_age_ms`** (NEW)         | 18,851.2 |
| 4  | **`inter_reconf_slope_ms`** (NEW) | 18,780.7 |
| 5  | **`drb_4_age_ms`** (NEW)         | 16,665.4 |
| 6  | `max_grant_bytes`                |  7,904.2 |
| 7  | `phy_snr_rx0_mean`               |  7,624.1 |
| 8  | `total_grant_bytes`              |  6,602.0 |
| 9  | `mean_grant_bytes`               |  6,580.3 |
| 10 | `phy_rsrp_mean`                  |  6,386.9 |
| 11 | `reconf_in_window`               |  6,025.0 |
| 12 | `pucch_sr_count`                 |  5,871.6 |
| 13 | `phy_rsrq_mean`                  |  5,636.1 |
| 14 | `time_into_activity_ms`          |  3,851.2 |
| 15 | **`drb_5_age_ms`** (NEW)         |  3,473.5 |

> **3 of the 10 new features ranked in the top 5** by gain immediately, and a 4th made the top 15. The temporal / recency features are real signal.

---

## 7. Honest Limitations

- **Data volume:** Only 331 DRB-setup events across 39,916 windows — too few for reliable multi-output multiclass prediction (random baseline = 33% for 3-class).
- **Templated configs:** The eNB sends mostly canonical DRB parameter sets (carrier defaults). Most of the "prediction" achievable is a lookup table of standard configs, not a learned mapping from radio state.
- **Binary baseline:** `out_has_reconf` AUC-PR ~0.30 is below the threshold needed for publishable results. Need to push this to **0.50+** before extending to content prediction is credible.
- **PUCCH gaps:** `pucch_ack_count` and `pucch_nack_count` still zero — no Format 1A/1B traffic in any of the 9 captures.
- **RLC dynamics gap:** Snapdragon 8 Gen 2 binary RLC Config format is undecodable; retx counts, NACK counts, poll events remain placeholder zeros. Currently bypassed via RRC-XML state tracking.
- **Practical motive:** Original motivation (UE pre-warming) doesn't strongly require 500 ms ahead; actual reconfig latency is microseconds. Research framing may need tightening.

---

## 8. Recommendations

| Option | Action | Effort | Expected outcome |
|--------|--------|--------|------------------|
| **A. Narrow scope** | Predict reconfig **type** (4-class: meas / DRB-add / PHY / HO) instead of full 25-output content. | Low — uses existing dataset; 6,867 positives available immediately. | Tractable; demonstrates new temporal features genuinely help; professor-defensible result. |
| **B. More targeted captures** | Run scripted bearer establish / release cycles: forced VoLTE drops, repeated data toggling, signal-attenuated handover triggers. | Medium — ~1 week of lab time. | Only path to making the full 25-output content prediction statistically valid (>1000 DRB events). |
| **C. Fix binary baseline** | Push `out_has_reconf` AUC-PR from 0.30 → 0.50+. Feature pruning, leakage audit on `time_since_last_reconf_ms`, possibly split-by-time instead of split-by-batch. | Low–Medium. | Establishes credible baseline before reaching for full content prediction. |
| **D. Hierarchical model** | Two-stage: predict `has_reconf` first, then conditional on positives predict params. | Low. | Concentrates learning capacity on the rare-event subset; may unstick multiclass models. |

**Suggested next week:** pursue **A** and **C** in parallel using the current dataset. Begin **B** if lab time permits.

---

## 9. Output Files

| Type        | Path (under `/home/venu/Desktop/ueautomation/`) | Description |
|-------------|------------------------------------------------|-------------|
| Parser      | `scripts/parse_drb_contents.py`                 | `.mi2log` → 82-col CSV/xlsx |
| Trainer     | `scripts/train_drb_contents.py`                 | Multi-output LightGBM |
| Combined    | `outputs/drb_contents_combined.parquet`         | 39,916 × 82, all 9 batches |
| Per-batch   | `outputs/batches/batch_{1..10}_*.csv`           | Individual batch CSVs |
| Models      | `outputs/drb_contents_models.pkl`               | All trained LightGBM models |
| Models txt  | `outputs/model_out_*.txt`                       | Human-readable model dumps |
| Inspect     | `outputs/drb_features_phase_d1_82col.xlsx`      | Full Phase_D_Batch_1 (6,262 rows) |
| Inspect     | `outputs/drb_features_reconf_only.xlsx`         | All reconf windows (6,867 rows) |
| Inspect     | `outputs/drb_features_drb_setup_only.xlsx`      | Only DRB-setup label rows (**331** — primary attachment for professor) |
| Importance  | `outputs/drb_contents_feature_importance.csv`   | Feature gain ranking |
| This file   | `outputs/drb_weekly_summary_may14.md`           | ← This summary |

