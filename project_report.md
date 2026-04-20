# Smart RRC Reconfiguration Predictor — Project Report

> **Device:** OnePlus 11R (CPH2487) — Qualcomm Snapdragon 8 Gen 2  
> **Network:** Airtel 4G LTE (MCC=404, MNC=10), Band 1 + Band 8 CA  
> **Tool:** MobileInsight 6.0.0 (custom-patched C++ decoder)  
> **Environment:** Python 3.12, `mi-env` virtualenv, Ubuntu 22.04  
> **Date:** April 2026

---

## Table of Contents

1. [Project Goal](#1-project-goal)
2. [Why This Problem Is Hard](#2-why-this-problem-is-hard)
3. [End-to-End Data Collection Pipeline](#3-end-to-end-data-collection-pipeline)
4. [MobileInsight C++ Decoder — What We Fixed](#4-mobileinsight-c-decoder--what-we-fixed)
5. [PHY Decoder Reverse Engineering (Recent Work)](#5-phy-decoder-reverse-engineering-recent-work)
6. [Dataset Column Design — All 49 Columns and Why](#6-dataset-column-design--all-49-columns-and-why)
7. [Dataset Summary — 13 Batches Across 3 Phases](#7-dataset-summary--13-batches-across-3-phases)
8. [ML Training Strategy](#8-ml-training-strategy)
9. [Current Status and Next Steps](#9-current-status-and-next-steps)
10. [File Map](#10-file-map)

---

## 1. Project Goal

Train a machine learning model that predicts **RRC Reconfiguration messages before they arrive** — predicting both **when** a reconfiguration will occur (within the next 500ms) and **what** it will contain (handover? carrier aggregation? measurement config update?).

### Why Predict RRC Reconfigurations?

RRC (Radio Resource Control) Reconfiguration is the LTE control plane's primary mechanism for changing the UE's radio configuration. It carries:

- **Handover commands** — switches serving cell, causes ~50-200ms service interruption
- **Carrier Aggregation changes** — activates/deactivates secondary cells (SCells) for higher throughput
- **Measurement configuration updates** — tells the UE which frequencies/cells to monitor
- **DRB Setup/Release** — adds or removes data radio bearers
- **PHY/MAC configuration** — tunes MIMO, DRX, CSI reporting

A model that predicts these events 500ms ahead enables:
- **Proactive handover preparation** — pre-load neighbor cell parameters
- **Battery optimization** — keep modem awake only when a reconfig is imminent
- **QoS-aware scheduling** — pre-allocate resources before throughput spikes

---

## 2. Why This Problem Is Hard

### 2.1 The Data Access Problem

Standard Android APIs (`TelephonyManager`) expose only coarse, heavily averaged signal metrics (RSRP, RSRQ, SINR) at ~1-second granularity. They do not expose:
- MAC-layer scheduling (grant sizes, buffer status reports)
- HARQ retransmission activity
- Precise RRC message timing (which reconfig contained what)
- Neighbor cell measurements at the resolution the eNB sees

To get this data, we need access to the **Qualcomm Diagnostic Monitor (DM) interface** — a proprietary serial protocol that streams raw baseband telemetry.

### 2.2 The Decoder Problem

MobileInsight 6.0.0 includes C++ decoders for Qualcomm DM packets. But MobileInsight was designed for older chipsets (Snapdragon 820/835 era). The Snapdragon 8 Gen 2 sends newer subpacket versions with shifted binary field layouts that MobileInsight's existing decoders cannot handle — they silently produce zeros or garbage values.

We had to reverse-engineer the correct byte offsets ourselves (see Section 5).

### 2.3 The Sparsity Problem

The LTE protocol uses **DRX (Discontinuous Reception)** to save battery — the modem sleeps 310ms out of every 320ms (long DRX cycle). During sleep, there are zero MAC events. A naive 100ms sliding window approach produces mostly empty feature vectors that tell the model nothing.

---

## 3. End-to-End Data Collection Pipeline

```
┌─────────────────┐    USB Serial (115200 baud)    ┌──────────────────────┐
│  OnePlus 11R     │ ─────────────────────────────► │  dm_capture.py       │
│  Snapdragon 8G2  │   Raw Qualcomm DM packets      │  (MobileInsight)     │
│  /dev/ttyUSB0    │                                │  → .bin (raw bytes)  │
│  USB diag mode   │                                │  → .mi2log (decoded) │
└─────────────────┘                                └──────────┬───────────┘
                                                              │
                                                              ▼
                                                   ┌──────────────────────┐
                                                   │  parse_logs_ml.py    │
                                                   │  (Python, --phy flag)│
                                                   │  → 49-column CSV     │
                                                   └──────────┬───────────┘
                                                              │
                                    ┌─────────────────────────┼──────────────────────────┐
                                    ▼                         ▼                          ▼
                           ┌─────────────────┐    ┌─────────────────┐    ┌──────────────────────┐
                           │ MAC/RRC/NAS rows │    │  PHY_SERV rows  │    │  PHY_NEIGH rows      │
                           │ (every ~1-10ms  │    │  (every ~40ms)  │    │  (every ~200-320ms)  │
                           │  during active) │    │  RSRP/RSRQ/SNR  │    │  Neighbor cell RSRP  │
                           └────────┬────────┘    └────────┬────────┘    └──────────┬───────────┘
                                    │                      │                         │
                                    └──────────────────────┴─────────────────────────┘
                                                           │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │   pipeline.py   │
                                                  │  (to be built)  │
                                                  │  250ms windows  │
                                                  │  50ms stride    │
                                                  │  label: reconf  │
                                                  │  in next 500ms? │
                                                  └────────┬────────┘
                                                           │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │   train.py      │
                                                  │  (to be built)  │
                                                  │  LightGBM       │
                                                  │  LOBO-CV        │
                                                  └─────────────────┘
```

### 3.1 Phone Setup

USB diagnostic mode is unlocked by running on the phone (as root via ADB):

```bash
setprop sys.usb.config diag,adb
```

This exposes a Qualcomm DM serial port at `/dev/ttyUSB0` on the Linux PC.

### 3.2 What dm_capture.py Does

1. Opens `/dev/ttyUSB0` at 115200 baud via `pyserial`
2. Sends log filter commands via `dm_collector_c.generate_log_config()` to tell the modem which log types to emit
3. Reads raw bytes, decodes via `dm_collector_c.feed_binary()` (C++ extension)
4. Writes decoded packets to `.mi2log` binary format AND raw bytes to `.bin`

### 3.3 The 16 Log Types We Capture

| Log Type | What It Gives Us |
|----------|-----------------|
| `LTE_RRC_OTA_Packet` | RRC messages: reconfigs, meas reports, paging, SIBs |
| `LTE_RRC_Serv_Cell_Info` | Serving cell EARFCN, PCI, band |
| `LTE_MAC_UL_Transport_Block` | Uplink grant sizes, BSR, HARQ, LCID assignments |
| `LTE_MAC_DL_Transport_Block` | Downlink grant sizes, LCID assignments |
| `LTE_NAS_ESM_OTA_Incoming/Outgoing` | Bearer setup/teardown events |
| `LTE_NAS_EMM_OTA_Incoming/Outgoing` | Attach, TAU, authentication |
| `LTE_PDCP_DL/UL_Ctrl_PDU` | PDCP control (compression setup) |
| `LTE_RLC_DL/UL_AM_All_PDU` | RLC acknowledged mode PDUs |
| `LTE_PHY_Serv_Cell_Measurement` | **RSRP, RSRQ, SNR** — primary signal metrics |
| `LTE_PHY_PUSCH_CSF` | **CQI, Rank Index** — channel quality for MIMO |
| `LTE_PHY_Connected_Mode_Intra_Freq_Meas` | **Neighbor cell RSRP** — handover trigger |

### 3.4 The .mi2log Format — Critical Gotcha

The `.mi2log` file stores packets **already decoded by the C++ engine at capture time**. If you recompile the C++ decoder and try to re-parse an old `.mi2log`, you still get the old wrong values — the decoding happened when the file was written, not when it is read. **New captures are required after any decoder fix.**

---

## 4. MobileInsight C++ Decoder — What We Fixed

The Snapdragon 8 Gen 2 sends newer subpacket versions than MobileInsight 6.0.0 supports. Without patches, the decoder either silently drops packets or outputs zeros/garbage. We made 5 patches:

### Patch 1: LTE RRC OTA — Version 27
- **File:** `dm_collector_c/log_packet.cpp` ~line 312
- **Symptom:** `(MI)Unknown LTE RRC version: 0x1b` in output, RRC packets dropped
- **Fix:** `case 27:` fallthrough to `case 25:` decoder (layouts are compatible)

### Patch 2: MAC UL/DL Transport Block — Versions 0x30, 0x32
- **File:** `dm_collector_c/log_packet.cpp` ~line 11854
- **Symptom:** MAC packets crashed the decoder
- **Fix:** `case 0x30:` and `case 0x32:` fallthrough to `case 0x28:` decoder

### Patch 3: PHY Intra-Freq Meas — Version 0x38 (56)
- **File:** `dm_collector_c/log_packet.cpp` ~line 1725
- **Symptom:** Neighbor cell measurements silently dropped
- **Fix:** `case 0x38:` fallthrough to nearest compatible layout

### Patch 4: PHY PUSCH CSF (CQI/Rank) — Version 0xa3 (163)
- **File:** `dm_collector_c/lte_phy_pusch_csf.h` ~line 173
- **Symptom:** CQI and Rank Index always zero
- **Fix:** `case 0xa3:` fallthrough to existing CQI decoder
- **Result:** CQI values 5-7, Rank 1-2 now extracted correctly

### Patch 5: PHY Serving Cell Measurement — Version 59 (0x3B)
- **File:** `dm_collector_c/log_packet.cpp` ~line 3144
- **Symptom:** RSRP = -173 dBm, RSRQ = +9 dB (completely wrong)
- **Why fallthrough failed:** The Snapdragon 8 Gen 2 firmware shifted field positions — no legacy layout is compatible
- **Fix:** Custom byte-level decoder with reverse-engineered offsets (see Section 5)
- **Result:** RSRP = -97.12 dBm, RSRQ = -17.00 dB ✅

### Build Command (after any C++ change)

```bash
cd /home/venu/Downloads/MobileInsight-6.0.0
source mi-env/bin/activate
python3 setup.py build && python3 setup.py install
# CRITICAL: copy .so to source tree, or OfflineReplayer uses stale binary
cp mi-env/lib/python3.12/site-packages/mobile_insight/monitor/dm_collector/dm_collector_c.cpython-312-x86_64-linux-gnu.so \
   mobile_insight/monitor/dm_collector/
```

---

## 5. PHY Decoder Reverse Engineering (Recent Work)

This section documents the most technically complex part of the project — discovering the correct byte offsets for RSRP, RSRQ, and SNR in Qualcomm subpacket version 59.

### 5.1 The Problem

When MobileInsight decoded `LTE_PHY_Serv_Cell_Measurement` packets from the OnePlus 11R, it reported:
- RSRP = -173 dBm (the minimum possible value, essentially "not decoded")
- RSRQ = +9.6 dB (positive value, physically impossible in real networks)

Meanwhile, Android's `dumpsys telephony.registry` reported the true values:
- RSRP = -96 dBm
- RSRQ = -17 dB

The existing decoder was using binary field offsets from an older chipset's firmware that no longer matched the Snapdragon 8 Gen 2's layout.

### 5.2 Methodology

**Step 1 — Capture raw bytes.** Injected a `printf` hex dumper in the C++ decoder to print the raw subpacket bytes before any interpretation:

```cpp
// Temporary debug: dump first 100 bytes of subpacket
for (int i = 0; i < std::min(sp_len, 100); i++) {
    printf("%02X ", (unsigned char)b[offset + i]);
}
```

Captured 3 representative packets with known ground truth (ADB simultaneously running):
```
ADB ground truth: RSRP=-96 dBm, RSRQ=-17 dB
```

**Step 2 — Mathematical targeting.** Given the standard Qualcomm encoding formulas:
- RSRP: `raw_value * 0.0625 - 180.0 = dBm` → target raw for -96 dBm = **1344**
- RSRQ: `raw_value * 0.0625 - 30.0 = dB` → target raw for -17 dB = **208**

**Step 3 — Brute-force scan.** Wrote `/tmp/scan_targeted.py` — tried every byte offset (0–96) × every bit shift (0–23) × every bitmask combination, checking which produced values within ±5 dBm of the target.

**Step 4 — Cross-validation.** Found offsets that hit the target across ALL 3 packets simultaneously. Coincidental false positives would be inconsistent; the true offset must give similar values in all 3 packets.

### 5.3 Validated Offsets

| Metric | Byte Offset | Bit Shift | Bit Mask | Formula | Validated Output |
|--------|-------------|-----------|----------|---------|-----------------|
| **RSRP Rx[0]** | 44 | `>> 12` | `& 4095` | `val × 0.0625 − 180.0` dBm | **-97.12 dBm** (ADB: -96 ✅) |
| **RSRQ Rx[0]** | 34 | `>> 13` | `& 1023` | `val × 0.0625 − 30.0` dB | **-17.00 dB** (ADB: -17 ✅) |
| **SNR Rx[0]** | 32 | `>> 9` | `& 511` | `val × 0.1 − 20.0` dB | **10.70 dB** (plausible indoor ✅) |

### 5.4 The C++ Fix

```cpp
case 59: {
    // Custom decoder for version 59 (Snapdragon 8 Gen 2)
    // Offsets validated by scan_targeted.py against 3-packet ADB ground truth:
    //   ADB truth: RSRP=-96 dBm, RSRQ=-17 dB
    //   PKT1..3 → RSRP ~ -97 dBm, RSRQ ~ -17 dB (within ADB rounding)

    // RSRP Rx[0] — offset 44, shift 12, mask 4095
    unsigned int utemp_rsrp = *((unsigned int *) (b + offset + 44));
    float rsrp0_val = float((utemp_rsrp >> 12) & 4095) * 0.0625 - 180.0;

    // RSRQ Rx[0] — offset 34, shift 13, mask 1023
    unsigned int utemp_rsrq = *((unsigned int *) (b + offset + 34));
    float rsrq0_val = float((utemp_rsrq >> 13) & 1023) * 0.0625 - 30.0;

    // SNR (FTL) — offset 32, shift 9, mask 511
    unsigned int utemp_snr = *((unsigned int *) (b + offset + 32));
    float snr0_val = float((utemp_snr >> 9) & 511) * 0.1 - 20.0;

    // ... emit as PyObjects ...
    success = true;
    break;
}
```

**Key bug also fixed:** The original code had a use-after-decref (`pyfloat_rsrp` was Py_DECREF'd before being used to build the second tuple). This is a memory-safety bug — fixed by reordering the `Py_DECREF` calls.

### 5.5 Validation Tools Built

| Script | Purpose |
|--------|---------|
| `adb_signal_poller.py` | Polls `adb shell dumpsys telephony.registry` every 0.5s, saves RSRP/RSRQ/SINR to CSV as ground truth |
| `correlate_phy_offsets.py` | Replays `.mi2log` via OfflineReplayer, aligns packets to ADB timestamps (±2s), computes Pearson correlation for each candidate offset+shift — correct offset has r ≈ 1.0 |

---

## 6. Dataset Column Design — All 49 Columns and Why

The output CSV from `parse_logs_ml.py --phy --mac` has **49 columns**. Every column was deliberately chosen. Here is the rationale for each group.

### 6.1 Timing & Identity (3 columns)

| Column | Why |
|--------|-----|
| `Timestamp` | Modem-reported UTC timestamp (microsecond precision). Used for windowing, label assignment, activity join. More precise than system clock — the modem timestamps directly from baseband hardware. |
| `Layer` | Which protocol layer this row came from: `MAC_UL`, `MAC_DL`, `RRC`, `NAS_EMM`, `NAS_ESM`, `PHY_SERV`, `PHY_CSF`, `PHY_NEIGH`. The event stream is heterogeneous — layer is required to know which other columns are populated. |
| `Message_Type` | Finer classification within layer (e.g., `rrcConnectionReconfiguration` vs `measurementReport`). Used to compute `measurement_report_count` per window — a key predictor. |

### 6.2 MAC Layer Features (13 columns)

These come from `LTE_MAC_UL/DL_Transport_Block` — the MAC scheduler's per-transmission records. MAC is the most dense source: ~1 event per subframe (1ms) during active periods.

| Column | Rationale |
|--------|-----------|
| `Direction` | UL vs DL. Traffic patterns differ: heavy DL = downloading (SCell activation likely); heavy UL = upload/VoLTE (different bearer config). |
| `Grant_Bytes` | The MAC scheduler's allocation decision. Directly encodes network demand as the eNB sees it. A **sudden jump in grant_bytes is the #1 predictor of SCell activation** — the eNB activates carrier aggregation in response to sustained high demand. |
| `Num_Samples` | How many MAC sub-samples were bundled into this log packet. Used to normalize rates. |
| `LCIDs` | All Logical Channel IDs in this transport block (pipe-delimited). LCIDs encode the traffic type: LCID 3-10 = data bearers, LCID 1-2 = signaling bearers, special values like `S-BSR`, `L-BSR`, `Padding`, `PHR`. |
| `Data_LCIDs` | Only data-bearing LCIDs (3-10). Used to derive `data_lcid_bitmap` — a 5-bit vector showing which bearers are active. DRB-5 alone = VoLTE; DRB-3+4 = dual-carrier data. |
| `Has_SBSR` | Short Buffer Status Report present. A BSR means the UE is telling the eNB "I have data buffered that you haven't granted me bandwidth for." Rising BSR rate = demand building → eNB will respond with more grants or SCell activation. |
| `Has_LBSR` | Long Buffer Status Report — same signal but for larger buffer buildups. LBSR appears when the UE has significant backlog across multiple logical channels. |
| `Has_Padding` | Padding bytes were added to fill the MAC PDU. High padding = grant size exceeds actual data = the UE is being over-allocated relative to demand. Falling padding after high padding means real data started arriving. |
| `Has_PHR` | Power Headroom Report present. The UE reports how much uplink transmit power headroom it has. Low PHR = UE is near power limits, affects MIMO rank and MCS choices. |
| `HARQ_ID` | HARQ (Hybrid ARQ) process ID. HARQ enables stop-and-wait retransmission with parallelism. Multiple simultaneous HARQ IDs = high spatial multiplexing. Pattern of retransmissions indicates channel quality. |
| `Num_Data_LCIDs` | Count of distinct data bearers active. Grows when new bearers are added (DRB_Setup reconfig). Drops when bearers are released (DRB_Release reconfig). |
| `Num_Sig_LCIDs` | Count of signaling LCIDs (SRB1/SRB2 = LCID 1/2). A spike in signaling traffic precedes reconfigurations — the RRC messages themselves ride on these bearers. |

### 6.3 RRC Labels and Targets (13 columns)

These come from `LTE_RRC_OTA_Packet`. Most rows have `Is_RRC_Reconf=0`; the reconf rows carry the target flags.

| Column | Rationale |
|--------|-----------|
| `Is_RRC_Reconf` | **The primary training label.** 1 on rows that are `rrcConnectionReconfiguration` messages. Used to assign the `reconfig_in_next_500ms` label to preceding windows. |
| `Target_MeasConfig` | **Most frequent reconfig type (~75% of all reconfigs).** eNB updates measurement configuration — which frequencies to monitor, which thresholds trigger reports. Predictable from connection setup timing patterns. |
| `Target_Handover` | **High-impact event.** Cell change causes ~50-200ms service interruption. Predictable from rising neighbor RSRP + measurement report bursts. The `PHY_Neighbor_RSRP` column exists specifically to enable this prediction. |
| `Target_DRB_Setup` | Data Radio Bearer addition — new application started, or bearer re-establishment after failure. Correlated with NAS ESM events. |
| `Target_DRB_Release` | Data Radio Bearer teardown — application finished. Correlated with prior DRB_Setup and silence in grant bytes. |
| `Target_SCell_Config` | **Carrier Aggregation add/modify.** Activated when sustained high throughput demand requires secondary component carrier. Best predictor: `grant_bytes_delta` spike. |
| `Target_ENDC_Config` | 5G NR secondary cell configuration (EN-DC). Rare on this network (Airtel Band 1/8), included for completeness. |
| `Target_PHY_Config` | Physical layer parameter update (MIMO config, CSI reporting, SRS config). Correlated with CQI changes. |
| `Target_MAC_Config` | MAC-layer parameter update (DRX cycle changes, BSR timers). DRX config is consistent in our dataset — this target is rarely 1. |
| `Target_Security` | Security algorithm reconfiguration. Typically one-time during connection setup. |
| `Target_SRB_Setup` | Signaling Radio Bearer addition. Appears during connection setup and RRC re-establishment. |
| `Target_RLC_Config` | RLC (Radio Link Control) configuration update. Co-occurs with DRB changes. |
| `Target_PCI` | **Physical Cell ID of handover target** (integer, -1 if no handover). Enables the Stage 2 "predict which cell we're handing over to" capability. |

### 6.4 RRC Detail (1 column)

| Column | Rationale |
|--------|-----------|
| `RRC_Detail` | Pipe-delimited string listing all configuration components present in the reconf (e.g., `"MeasConfig\|MeasObj_EUTRA\|SCell_CA\|PHY(MIMO,CQI)"`). Used for deep analysis and debugging. Not a direct ML feature — too high-cardinality. |

### 6.5 NAS Features (2 columns)

These come from NAS (Non-Access Stratum) EMM and ESM packets — the upper protocol layer that manages attach/detach and bearer activation.

| Column | Rationale |
|--------|-----------|
| `NAS_EMM_Type` | EMM message type (Attach Request, TAU Request, Authentication Request, etc.). Network attach events are followed by a burst of RRC reconfigurations for bearer setup and measurement configuration. |
| `NAS_ESM_Type` | ESM message type (Activate Default Bearer, Activate Dedicated Bearer, Modify Bearer, Deactivate Bearer). Directly precedes `Target_DRB_Setup` and `Target_DRB_Release` reconfigurations — the NAS request causes the RRC response. |

### 6.6 PHY Serving Cell Features (10 columns)

These come from `LTE_PHY_Serv_Cell_Measurement` (custom-decoded via Patch 5 above). Rate: ~1 measurement per radio frame = every 40ms.

| Column | Rationale |
|--------|-----------|
| `PHY_RSRP` | **Reference Signal Received Power (dBm).** The primary serving cell signal strength. Absolute value matters less than its **trajectory** — falling RSRP over 500ms is the key handover trigger. Validated: -97.12 dBm on our device. |
| `PHY_Inst_RSRP` | Instantaneous RSRP (vs the filtered/averaged `PHY_RSRP`). Shows fast fading; high variance between `PHY_RSRP` and `PHY_Inst_RSRP` = deep fading = handover may be triggered. |
| `PHY_RSRQ` | **Reference Signal Received Quality (dB).** Signal quality accounting for interference. RSRQ degradation (becomes more negative) while RSRP stays constant = interference increased = reconfiguration to change measurement thresholds or activate carrier aggregation on a cleaner band. |
| `PHY_RSRQ_Rx0`, `PHY_RSRQ_Rx1` | Per-antenna RSRQ. Asymmetry between antennas indicates UE orientation change or multipath. Used to derive rank/MIMO decisions. |
| `PHY_SNR_Rx0`, `PHY_SNR_Rx1` | FTL (Frequency Tracking Loop) SNR per antenna. SNR is the primary driver of MCS (Modulation and Coding Scheme) selection. Falling SNR forces lower MCS → lower throughput → eNB may reconfigure to compensate. |
| `PHY_SFN` | System Frame Number (0-1023, wraps). With `PHY_SubFN`, gives absolute modem time at ~1ms resolution. Used to align PHY measurements with MAC events. |
| `PHY_SubFN` | Subframe Number (0-9 within each SFN). Together with SFN: `absolute_ms = SFN * 10 + SubFN`. |
| `PHY_Serving_Cell_Idx` | Whether this measurement is for the primary cell (PCell) or a secondary cell (SCell0, SCell1 for CA). SCell measurements appearing = carrier aggregation already active. |

### 6.7 PHY CQI / Rank (3 columns)

These come from `LTE_PHY_PUSCH_CSF` (Channel State Feedback via PUSCH) — the UE's channel quality report to the eNB.

| Column | Rationale |
|--------|-----------|
| `PHY_CQI` | **Channel Quality Indicator (0-15).** The UE's recommendation for which MCS (modulation and coding scheme) the eNB should use. CQI=15 = excellent channel, 64-QAM, high MIMO rank. CQI=0 = terrible channel. **CQI is the single best predictor of `Target_PHY_Config`** — the eNB reconfigures CQI reporting parameters when channel quality is chronically poor or changes type. |
| `PHY_Rank` | **Spatial Rank (1 or 2).** Rank 1 = single MIMO stream (beamforming), Rank 2 = spatial multiplexing (2 parallel streams). A rank change from 2→1 means channel conditions degraded and MIMO can no longer be exploited. This often co-occurs with `Target_PHY_Config` reconfigs that update antenna configuration. |
| `PHY_Carrier` | Whether this CSF report is for the Primary Component Carrier (PCC) or Secondary (SCC). SCC CSF appearing = CA is active; disappearing = CA deactivated. |

### 6.8 PHY Neighbor Cell Features (4 columns)

These come from `LTE_PHY_Connected_Mode_Intra_Freq_Meas` — the UE's A3/A5 event measurement reports.

| Column | Rationale |
|--------|-----------|
| `PHY_Neighbor_RSRP` | **RSRP of the strongest neighbor cell (dBm).** This is the critical handover trigger: when neighbor RSRP exceeds serving cell RSRP by the A3 offset (typically 3 dB), the UE sends a MeasurementReport and the eNB initiates handover. Computing `PHY_Neighbor_RSRP - PHY_RSRP` gives the A3 condition directly. |
| `PHY_Neighbor_RSRQ` | RSRQ of the strongest neighbor. Used with A5 events (conditional handover based on both serving and neighbor quality). |
| `PHY_Neighbor_PCI` | Physical Cell ID of the strongest neighbor. Together with `Target_PCI`, enables predicting not just *that* a handover will happen but *which cell* it will target. |
| `PHY_Num_Neighbors` | Total number of neighbors being measured. Rising neighbor count = UE is in a dense cell area or is moving. High neighbor counts correlate with more frequent handovers. |

---

## 7. Dataset Summary — 13 Batches Across 3 Phases

### Phase A — Baseline Single-Scenario (4 batches)

Individual scenarios run in isolation. Each batch captures one activity type cleanly.

| Batch | Scenario | Duration | Approx Reconfigs |
|-------|----------|----------|-----------------|
| A1 | Browsing | ~60s | ~43 |
| A2 | VoiceCall | ~90s | ~24 |
| A3 | DataToggle | ~30s | ~30 |
| A4 | UploadStress | ~160s | ~103 |

**Purpose:** Establish clean per-scenario baselines. Each scenario has a characteristic reconfig signature:
- VoiceCall: ~0.24 reconf/sec, dominated by MeasConfig + DRB_Setup
- Browsing: ~0.71 reconf/sec, dominated by MeasConfig bursts
- UploadStress: SCell activation visible, sustained high UL grant bytes

### Phase B — Idle-to-Active Transitions (4 batches)

Captures the moment the UE transitions from idle/DRX to active data transfer.

| Batch | Scenario | Duration | Key Feature |
|-------|----------|----------|------------|
| B1-B4 | Browsing (from idle) | ~60s each | First 5-10 seconds = connection establishment + burst of setup reconfigs |

**Purpose:** Profile the "wakeup burst" — a cluster of 3-5 rapid reconfigs that always appears when resuming from idle. The model must learn this pattern without overfitting to it.

### Phase C — Chaos / Mixed Activities (5 batches)

19 overlapping activities run in automated sequences. The most realistic and most valuable data.

| Batch | Duration | Reconfigs | Scenarios |
|-------|----------|-----------|-----------|
| C1 | ~27 min | ~140 | Mixed: Browsing + VoiceCall + DataToggle |
| C2 | ~27 min | ~860 | Mixed: Download + UploadStress + YouTube |
| C3 | ~27 min | ~750 | Full Stress: all scenarios overlapping |
| C4 | ~27 min | ~600 | Mixed: GuardBand transitions |
| C5 | ~27 min | ~500 | Mixed: held out as test set |

**Total positive events across all 13 batches: ~3,471 RRC Reconfigurations**

### Key Observation: DRX Configuration Is Constant

Across all 13 batches, the DRX configuration is identical:

| Parameter | Value |
|-----------|-------|
| onDurationTimer | 10ms |
| drx-InactivityTimer | 100ms |
| longDRX-Cycle | 320ms |
| shortDRX-Cycle | 80ms |

This means during DRX long sleep, the modem is awake for only 10ms out of every 320ms — a 97% sleep ratio. Empty windows are not missing data; they are the modem sleeping.

---

## 8. ML Training Strategy

### 8.1 The Windowing Problem

The data is event-driven, not time-sampled. Multiple rows can share the same timestamp (modem flushes diagnostic buffers in batches). A 100ms window sitting in a DRX long sleep period contains zero events — useless.

**Solution: 250ms windows with 50ms stride.**

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Window size | 250ms | Spans ≥1 DRX short cycle (80ms). Guarantees at least 1 on-duration event during active sessions. |
| Stride | 50ms | 80% overlap. 20 samples/sec. Sufficient temporal resolution. |
| Label horizon | 500ms forward | Enough lead time for proactive action. |
| Estimated windows (Phase C, 27 min) | ~32,400 | Sufficient training volume. |

**Label:** `reconfig_in_next_500ms` — binary 1 if any `Is_RRC_Reconf=1` row exists in the 500ms after the window ends.

### 8.2 Feature Engineering (30+ features per window)

All features are computed from the 49-column CSV. The most important features:

**Rate-of-change features (critical — the eNB responds to changing conditions, not absolute values):**
- `grant_bytes_delta` — current window grant sum minus previous window: sudden jump = download starting = SCell activation coming
- `measurement_report_acceleration` — 2nd derivative of measurement report rate: burst of reports after silence = handover imminent
- `bsr_rate_change` — rising BSR = buffer filling faster than grants = demand building
- `rsrp_delta` — RSRP trend: falling = handover trigger building
- `rsrp_vs_neighbor_delta` — serving RSRP minus strongest neighbor RSRP: approaching 0 dB = A3 event imminent = handover

**DRX-aware features:**
- `time_since_last_mac_ms` — ms since last MAC event: >100ms = in DRX sleep
- `drx_cycle_phase` — estimated state: `active`, `short_sleep`, `long_sleep`

### 8.3 Class Balance

With 250ms windows at 50ms stride and ~3,471 reconfigs:
- Positive windows: ~3,000-4,000 (each reconfig creates ~10 positive windows within 500ms horizon)
- Class ratio: ~8:1 negative:positive

**Strategy:** `is_unbalance=True` in LightGBM (native handling). For TCN: `pos_weight` in `BCEWithLogitsLoss`. Avoid SMOTE — it creates phantom patterns by interpolating between time-series points from different temporal contexts.

### 8.4 Model Architecture

**Stage 1: LightGBM (start here)**
- Trains in seconds
- Feature importance reveals which of the 30+ features actually matter
- ~200KB model, <1ms inference on ARM
- Expected: AUC-PR 0.75-0.85

```python
params = {
    'objective': 'binary',
    'metric': 'average_precision',
    'is_unbalance': True,
    'num_leaves': 63,
    'feature_fraction': 0.7,
    'learning_rate': 0.05,
    'num_iterations': 500,
    'early_stopping_rounds': 50,
}
```

**Stage 2: TCN (Temporal Convolutional Network)**
- 3 dilated causal conv layers (dilation 1, 2, 4)
- Input: last 20 windows (5 seconds of context)
- Effective receptive field: 3.75 seconds — covers DRX cycles and reconfig bursts
- ~50K parameters, ~200KB float16, 2-4ms inference
- Better than LSTM for our data size: parallel training, lower overfitting risk

**Stage 3: Ensemble**
- `P_final = alpha * P_lightgbm + (1-alpha) * P_tcn`
- LightGBM catches feature threshold patterns; TCN catches temporal sequences

### 8.5 Evaluation

**Primary metric: AUC-PR** (not AUC-ROC — misleadingly high in imbalanced settings)

**Cross-validation: Leave-One-Batch-Out (LOBO)**
- Each fold holds out one complete batch
- 11 folds (Phase A+B+C batches 1-4 as train, C5 as final test)

**Train/val/test split:**
| Split | Data |
|-------|------|
| Train | Phase A (4) + Phase B (4) + Phase C Batches 1-3 |
| Validation | Phase C Batch 4 |
| Test | Phase C Batch 5 (held out, never touched during development) |

**Per-scenario evaluation:** Report AUC-PR separately for each scenario type. High variance across scenarios = model is overfitting to batch-specific patterns.

### 8.6 Stage 2: Predicting Reconfiguration Content

After predicting *that* a reconfig is coming, predict *what* it will contain. Multi-label classifier with 7 sigmoid outputs corresponding to the `Target_*` columns.

| Reconfig Type | Feasibility | Primary Predictor |
|---------------|-------------|-------------------|
| MeasConfig | High | Connection timing, measurement report rate |
| Handover (occurrence) | Medium-High | Measurement report burst, RSRP delta |
| Handover (target PCI) | Low | Requires sustained neighbor RSRP tracking → PHY_Neighbor data |
| DRB_Setup | Medium | NAS ESM events |
| SCell_Config | Medium | grant_bytes_delta spike |
| PHY_Config | Low | CQI trend over time |

---

## 9. Current Status and Next Steps

### What Is Done

| Item | Status |
|------|--------|
| Phone USB diag setup | ✅ Complete |
| MobileInsight C++ patches (all 5) | ✅ Complete, compiled Apr 15 |
| PHY decoder reverse-engineering (Version 59) | ✅ Validated: RSRP=-97.12 ✅, RSRQ=-17.00 ✅ |
| parse_logs_ml.py (49-column CSV extraction) | ✅ Complete, PHY code ready with --phy flag |
| adb_signal_poller.py (ground truth tool) | ✅ Complete |
| correlate_phy_offsets.py (validation tool) | ✅ Complete |
| Phase A/B/C data collection (13 batches) | ✅ ~3,471 positive events |
| ML strategy design | ✅ Documented |

### What Is Next

| Step | Task | Details |
|------|------|---------|
| 1 | **Verify PHY in CSV** | Run 30s test capture with fixed decoder, parse with `--phy`, confirm PHY_RSRP ≈ -97, PHY_RSRQ ≈ -17 in output |
| 2 | **Phase D recapture** | Re-run Phase C chaos scenarios (existing mi2logs have PHY baked in with old wrong decoder — new captures needed for correct PHY data) |
| 3 | **Build `pipeline.py`** | 250ms windowing, label assignment (`reconfig_in_next_500ms`), activity ground truth join from labels.csv |
| 4 | **Build `features.py`** | 30+ features per window including rate-of-change deltas and PHY metrics |
| 5 | **Build `train.py`** | LightGBM baseline with LOBO-CV, feature importance analysis |
| 6 | **PHY impact analysis** | Retrain with/without PHY features to quantify AUC-PR improvement |

### Known Limitations

- **Single device, single network, single location** — model will be specific to OnePlus 11R + Airtel + Hyderabad cell sites (PCIs 84/86/87)
- **No per-subframe instantaneous RSRP** — the RSRP at offset 44 is a filtered average. Per-subframe RSRP at other offsets not yet mapped
- **OfflineReplayer hangs on large files** — keep captures ≤15 seconds; Phase C used automated scripting to handle this
- **PHY data requires new captures** — existing Phase A/B/C mi2logs decoded with old broken offsets, PHY values are wrong in those files

---

## 10. File Map

```
~/Downloads/MobileInsight-6.0.0/
├── mi-env/                                  # Python 3.12 virtualenv (DO NOT delete)
│   └── lib/python3.12/site-packages/
│       └── mobile_insight/monitor/dm_collector/
│           └── dm_collector_c.cpython-312.so  ← compiled decoder (built Apr 15 18:31)
├── dm_collector_c/
│   ├── log_packet.cpp                       ★ All 5 decoder patches (esp. case 59 @ line 3144)
│   ├── lte_phy_pusch_csf.h                  ★ CQI version switch (case 0xa3 patch)
│   └── ...
├── mobile_insight/monitor/dm_collector/
│   └── dm_collector_c.cpython-312.so        ★ MUST match build (copy after compile)
└── scripts/
    └── dm_capture.py                        ★ Live USB capture script

~/Desktop/ueautomation/
├── scripts/
│   ├── parse_logs_ml.py                     ★ Raw log → 49-column CSV (--phy --mac flags)
│   ├── adb_signal_poller.py                 Ground truth poller (ADB RSRP/RSRQ → CSV)
│   ├── correlate_phy_offsets.py             PHY offset validator (Pearson correlation)
│   ├── analyze_logs.py                      Activity label annotator
│   ├── pipeline.py                          ← TO BUILD: windowing + labeling
│   ├── features.py                          ← TO BUILD: feature engineering
│   └── train.py                             ← TO BUILD: LightGBM LOBO-CV
├── rrc_reconf-analysis/
│   ├── snapdragon_8Gen2_phy_offsets.md      ★ Full PHY reverse-engineering reference
│   ├── project_report.md                    ← This file
│   └── logs/                                Captured log copies
└── logs/
    ├── 20260325_*/                          Phase A+B batch sessions
    │   ├── scenario_*.mi2log               Raw decoded capture (old decoder — PHY wrong)
    │   ├── scenario_*.csv                  31-col CSV (no PHY — old captures)
    │   ├── scenario_*.bin                  Raw DM bytes
    │   └── labels.csv                      Activity ground truth timestamps
    └── 20260407_*/                          Phase C batch sessions (same structure)
```

---

*Report generated April 2026. For decoder details see `snapdragon_8Gen2_phy_offsets.md`. For ML strategy detail see the ML Training Strategy document.*
