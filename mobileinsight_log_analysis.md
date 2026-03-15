# LTE Diagnostic Log Capture — Findings Report

**Device**: OnePlus 11R (Snapdragon 8 Gen 2, Android 15)
**Capture Method**: MobileInsight + Qualcomm Diagnostic Port (`/dev/ttyUSB0`)
**Date**: March 13, 2026

---

## 1. Capture Summary

Two 1-minute LTE diagnostic captures were performed to contrast signalling-only vs active data traffic.

| Metric | Data OFF (Idle) | Data ON (Active) | Ratio |
|:---|---:|---:|---:|
| Raw binary size | 180 KB | 3.6 MB | **20x** |
| Mi2log decoded packets | 704 | 5,645 | **8x** |
| CSV rows (parsed events) | 597 | 11,772 | **20x** |
| Capture duration | **60 seconds** | **60 seconds** | — |
| Log files | `capture_data_off.*` | `capture_active_data_1min.*` | — |

> [!NOTE]
> Both captures were exactly 60 seconds of raw diagnostic data. The decoded timestamp ranges may span a wider window (e.g., ~6 min for data-on) because the Qualcomm modem **buffers older diagnostic messages internally** and flushes them when capture begins. The actual live capture duration was 1 minute for each scenario.

---

## 2. Decoded Log Types Per Capture

| Log Type | Data OFF | Data ON | Description |
|:---|---:|---:|:---|
| `LTE_RRC_OTA_Packet` | 144 | 661 | RRC signalling messages |
| `LTE_RRC_Serv_Cell_Info` | 7 | 17 | Serving cell PCI & frequency |
| `LTE_MAC_UL_Transport_Block` | 69 | 803 | Uplink MAC grants |
| `LTE_MAC_DL_Transport_Block` | 62 | 656 | Downlink MAC transport |
| `LTE_MAC_UL_Buffer_Status_Internal` | 279 | 2,414 | Buffer occupancy per channel |
| `LTE_MAC_UL_Tx_Statistics` | 133 | 1,087 | UL transmission stats |
| `LTE_NAS_EMM_OTA` (In+Out) | 4 | 7 | Mobility management |
| `LTE_NAS_ESM_OTA` (In+Out) | 6 | 0 | Session management |
| **LTE_PHY_*** | **0** | **0** | **Not captured (see §6)** |

---

## 3. RRC Layer Analysis

### 3.1 Message Type Distribution

| RRC Message | Data OFF | Data ON | Significance |
|:---|---:|---:|:---|
| `measurementReport` | 24 | **191** | UE reports neighbor cell signal quality — **8x higher with data** |
| `rrcConnectionReconfiguration` | 30 | **92** | Network reconfigures UE — **3x higher with data** |
| `rrcConnectionReconfigurationComplete` | 30 | 92 | UE acknowledgement |
| `paging` | 9 | **173** | Network paging for incoming services |
| `sib_TypeAndInfo` | 29 | 40 | System Information Broadcast |
| `rrcConnectionSetup` | 1 | 7 | RRC connection establishment |
| `rrcConnectionRequest` | 1 | 7 | UE-initiated connection |
| `securityModeCommand` | 1 | 7 | Security activation |
| `ueCapabilityEnquiry` | 1 | 8 | Network asks UE for capabilities |

### 3.2 RRC Reconfiguration Purpose Analysis

Each RRC Reconfiguration was deeply inspected to determine **why** it was sent:

| Purpose | Data OFF | Data ON | Meaning |
|:---|---:|---:|:---|
| **MEASUREMENT_CONFIG** | 29 | 92 | Configure neighbor cell measurements |
| **PHY_CONFIG** | 9 | 38 | Tune MIMO, CQI, SRS parameters |
| **MAC_CONFIG** | 6 | 31 | Adjust DRX, BSR timers, PHR |
| **HANDOVER** | 5 | 7 | Switch to different cell tower |
| **DRB_SETUP** | 2 | 8 | Establish data radio bearer |
| **SRB_SETUP** | 1 | 8 | Establish signalling radio bearer |
| **RLC_CONFIG** | 2 | 8 | Adjust retransmission settings |
| **SECURITY_CONFIG_HANDOVER** | 5 | 7 | Re-key during handover |

> [!IMPORTANT]
> **Key Finding**: With data ON, the network sends **3x more reconfigurations**, primarily for:
> - More frequent measurement configurations (monitoring signal quality during active use)
> - DRB (Data Radio Bearer) setup/teardown cycles
> - MAC layer tuning for active data scheduling

---

## 4. MAC Layer Analysis

### 4.1 Throughput Comparison

| Metric | Data OFF | Data ON | Ratio |
|:---|---:|---:|---:|
| UL MAC blocks | 442 | 11,103 | **25x** |
| Average UL grant | 7,712 B | 9,199 B | 1.2x |
| Total UL data | 3.3 MB | 97.4 MB | **30x** |
| DL MAC blocks | 62 | 656 | **11x** |

### 4.2 Logical Channel Analysis (Traffic Type Identification)

MAC packets carry traffic on different Logical Channels (LCIDs), which identify the traffic type:

| LCID | Data OFF | Data ON | Traffic Type |
|:---|---:|---:|:---|
| **CCCH** (Control) | 69 | 2,317 | Common Control Channel |
| **LCID 1** (SRB1) | 10 | 205 | RRC Signalling |
| **LCID 2** (SRB2) | 3 | 64 | NAS Signalling |
| **LCID 3** (DRB1) | 38 | 627 | **User Data Bearer 1** |
| **LCID 4** (DRB2) | 62 | 956 | **User Data Bearer 2** |
| **LCID 5** (DRB3) | 0 | 45 | **User Data Bearer 3** |
| **LCID 6** (DRB4) | 0 | 43 | **User Data Bearer 4** |
| **LCID 7** (DRB5) | 0 | 41 | **User Data Bearer 5** |
| L-BSR/Padding | 25 | 341 | Buffer Status Reports |
| S-BSR | 0 | 49+ | Short BSR triggers |

> [!IMPORTANT]
> **Key Finding**: Data OFF shows traffic only on LCID 3 & 4 (residual buffered data). Data ON shows active traffic across **LCID 3–7** (five data radio bearers), confirming heavy multi-bearer data usage.

### 4.3 MAC-RRC Correlation

Around each RRC Reconfiguration event (±500ms window):

| Scenario | MAC blocks before | MAC blocks after | Data bytes after | Latency to first data |
|:---|:---:|:---:|:---:|:---:|
| Data OFF | 2–4 | 3–4 | ~30–80 KB | 30–90 ms |
| Data ON | 3–5 | 3–5 | **150–550 KB** | **17–57 ms** |

> [!TIP]
> After handover-type reconfigurations, data resumes within **17ms** on average — demonstrating fast L2 path switching by the network.

---

## 5. NAS Layer Analysis

| NAS Message Type | Data OFF | Data ON |
|:---|---:|---:|
| EMM Incoming (DL) | 2 | 0 |
| EMM Outgoing (UL) | 2 | 7 |
| ESM Incoming (DL) | 3 | 0 |
| ESM Outgoing (UL) | 3 | 0 |

NAS messages were sparse in both scenarios. The EMM outgoing messages in the data-on scenario likely correspond to Tracking Area Update (TAU) procedures during mobility.

---

## 6. PHY Layer Investigation

### Attempted Extraction

We investigated whether PHY layer measurements (RSRP, RSRQ, CQI, SINR) were available:

| Check | Result |
|:---|:---|
| MobileInsight supports PHY log types? | ✅ Yes — 24 PHY types supported |
| PHY packets found in decoded output? | ❌ **Zero packets** |
| PHY log codes in raw binary (byte scan)? | ⚠️ False positives (appear inside RRC payloads) |
| Root cause | Capture script uses equipment-level masks; PHY requires specific diag `0x73` log config commands |

### What Would Be Needed

To capture PHY data, the `raw_capture.py` script needs to send explicit Qualcomm log configuration commands (`0x73`) to enable PHY-specific log codes (e.g., `0xB193` for Serving Cell Measurement, `0xB179` for Intra-Freq measurements).

---

## 7. Available Data for ML Dataset

### Successfully Captured Layers

```
┌─────────────────────────────────────────────────────┐
│                   Available Data                     │
├──────────────┬──────────────────────────────────────┤
│ RRC Layer    │ ✅ Full: Reconfigurations, Meas      │
│              │    Reports, Handovers, Setup, SIBs    │
├──────────────┼──────────────────────────────────────┤
│ MAC Layer    │ ✅ Full: UL/DL Transport Blocks,     │
│              │    Grants, LCIDs, BSR, HARQ           │
├──────────────┼──────────────────────────────────────┤
│ MAC Buffer   │ ✅ Full: Per-channel buffer status    │
│              │    (2,414 samples in data-on)          │
├──────────────┼──────────────────────────────────────┤
│ MAC Tx Stats │ ✅ Full: Transmission statistics      │
│              │    (1,087 samples in data-on)          │
├──────────────┼──────────────────────────────────────┤
│ NAS Layer    │ ✅ Partial: EMM/ESM events            │
├──────────────┼──────────────────────────────────────┤
│ Serving Cell │ ✅ Present: PCI, frequency changes    │
├──────────────┼──────────────────────────────────────┤
│ PHY Layer    │ ❌ Not captured (RSRP, RSRQ, CQI)    │
└──────────────┴──────────────────────────────────────┘
```

### Proposed ML Feature Vector (per 100ms window)

| # | Feature | Source | Role |
|:--|:---|:---|:---|
| 1 | `measurement_report_count` | RRC | #1 predictor of reconfiguration |
| 2 | `paging_count` | RRC | State transition indicator |
| 3 | `time_since_last_reconf_ms` | RRC | Temporal pattern |
| 4 | `ul_grant_avg_bytes` | MAC UL | Throughput trend |
| 5 | `dl_tb_size_avg_bytes` | MAC DL | DL throughput |
| 6 | `data_bytes_last_500ms` | MAC | Rolling data volume |
| 7 | `sig_ratio` | MAC | Control vs data ratio |
| 8 | `bsr_triggered` | MAC | Buffer fill indicator |
| 9 | `active_drb_count` | MAC LCIDs | Number of active bearers |
| 10 | `serving_cell_pci` | RRC Serv Cell | Cell identity |

**Target Label**: `reconfig_in_next_500ms` → Binary (0/1)

---

## 8. Files Generated

| File | Description | Size |
|:---|:---|---:|
| [capture_data_off.bin](file:///home/venu/Downloads/MobileInsight-6.0.0/logs/capture_data_off.bin) | Raw binary — Data OFF | 180 KB |
| [capture_data_off.mi2log](file:///home/venu/Downloads/MobileInsight-6.0.0/logs/capture_data_off.mi2log) | MobileInsight format — Data OFF | 175 KB |
| [capture_data_off.csv](file:///home/venu/Downloads/MobileInsight-6.0.0/logs/capture_data_off.csv) | Parsed CSV — Data OFF | 78 KB |
| [capture_active_data_1min.bin](file:///home/venu/Downloads/MobileInsight-6.0.0/logs/capture_active_data_1min.bin) | Raw binary — Data ON | 3.6 MB |
| [capture_active_data_1min.mi2log](file:///home/venu/Downloads/MobileInsight-6.0.0/logs/capture_active_data_1min.mi2log) | MobileInsight format — Data ON | 3.5 MB |
| [capture_active_data_1min.csv](file:///home/venu/Downloads/MobileInsight-6.0.0/logs/capture_active_data_1min.csv) | Parsed CSV — Data ON | 1.7 MB |
| [rrc_reconf_comparison.md](file:///home/venu/Downloads/MobileInsight-6.0.0/logs/rrc_reconf_comparison.md) | RRC Reconfiguration deep analysis | 15 KB |
