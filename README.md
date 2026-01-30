# MAC and RLC Configuration Analysis - 4G LTE Scenarios

## Overview

This report analyzes **MAC-MainConfig** and **RLC-Config** parameters extracted from 4 different LTE scenarios captured via GSMTAP. The analysis includes parameter comparisons and 3GPP TS 36.331 specification ranges.

---

## Scenarios Analyzed

| Scenario | Duration | RRC Reconfiguration Messages | Key Characteristics |
|----------|----------|------------------------------|---------------------|
| **Voice Call** | ~100s | 24 | VoLTE with DRB-5 (UM RLC) |
| **100MB Download** | ~157s | 103 | High-throughput data session |
| **Browsing + Download** | ~126s | 90 | Bursty data traffic |
| **Voice + YouTube Live** | ~104s | 34 | Mixed voice/data (voice bearer not captured) |

---

## MAC-MainConfig Parameters

### DRX Configuration (Observed in All Scenarios)

| Parameter | Observed Value | 3GPP TS 36.331 Range | Purpose |
|-----------|----------------|----------------------|---------|
| **onDurationTimer** | psf10 (10 subframes) | psf1, psf2, psf3, psf4, psf5, psf6, psf8, psf10, psf20, psf30, psf40, psf50, psf60, psf80, psf100, psf200 | Duration UE stays awake after waking up |
| **drx-InactivityTimer** | psf100 (100 subframes) | psf1, psf2, psf3, psf4, psf5, psf6, psf8, psf10, psf20, psf30, psf40, psf50, psf60, psf80, psf100, psf200, psf300, psf500, psf750, psf1280, psf1920, psf2560, psf0-v1020 | Time to wait after last PDCCH before entering DRX |
| **drx-RetransmissionTimer** | psf8 (8 subframes) | psf1, psf2, psf4, psf6, psf8, psf16, psf24, psf33 | Max contiguous subframes for DL retransmission |
| **longDRX-CycleStartOffset** | sf320 (320ms) | sf10, sf20, sf32, sf40, sf64, sf80, sf128, sf160, sf256, sf320, sf512, sf640, sf1024, sf1280, sf2048, sf2560 | Long DRX cycle duration |
| **shortDRX-Cycle** | sf80 (80ms) | sf2, sf5, sf8, sf10, sf16, sf20, sf32, sf40, sf64, sf80, sf128, sf160, sf256, sf320, sf512, sf640 | Short DRX cycle duration |
| **drxShortCycleTimer** | 1 | 1..16 (shortDRX-Cycle units) | Duration of short DRX cycle before transitioning to long |

### Other MAC Parameters

| Parameter | Observed Value | 3GPP TS 36.331 Range |
|-----------|----------------|----------------------|
| **timeAlignmentTimerDedicated** | infinity | sf500, sf750, sf1280, sf1920, sf2560, sf5120, sf10240, infinity |
| **mac-ContentionResolutionTimer** | sf64 | sf8, sf16, sf24, sf32, sf40, sf48, sf56, sf64 |

---

## RLC Configuration Parameters

### RLC-AM (Acknowledged Mode) - Data Bearers (DRB-3, DRB-4)

| Parameter | Observed Value | 3GPP TS 36.331 Range | Purpose |
|-----------|----------------|----------------------|---------|
| **t-PollRetransmit** | ms80 | ms5, ms10, ms15, ms20, ms25, ms30, ms35, ms40, ms45, ms50, ms55, ms60, ms65, ms70, ms75, ms80, ms85, ms90, ms95, ms100, ms105, ms110, ms115, ms120, ms125, ms130, ms135, ms140, ms145, ms150, ms155, ms160, ms165, ms170, ms175, ms180, ms185, ms190, ms195, ms200, ms205, ms210, ms215, ms220, ms225, ms230, ms235, ms240, ms245, ms250, ms300, ms350, ms400, ms450, ms500, ms800-v1310, ms1000-v1310, ms2000-v1310, ms4000-v1310 | Timer for poll retransmission |
| **pollPDU** | p16 (DRB) / pInfinity (SRB) | p4, p8, p16, p32, p64, p128, p256, pInfinity | PDU count before poll trigger |
| **pollByte** | kBinfinity | kB25, kB50, kB75, kB100, kB125, kB250, kB375, kB500, kB750, kB1000, kB1250, kB1500, kB2000, kB3000, kBinfinity | Byte count before poll trigger |
| **maxRetxThreshold** | t32 | t1, t2, t3, t4, t6, t8, t16, t32 | Max retransmissions before RLC failure |
| **t-Reordering** | ms35 | ms0, ms5, ms10, ms15, ms20, ms25, ms30, ms35, ms40, ms45, ms50, ms55, ms60, ms65, ms70, ms75, ms80, ms85, ms90, ms95, ms100, ms110, ms120, ms130, ms140, ms150, ms160, ms170, ms180, ms190, ms200 | Timer for RLC PDU reordering |
| **t-StatusProhibit** | ms0 (SRB) / ms25 (DRB) | ms0, ms5, ms10, ms15, ms20, ms25, ms30, ms35, ms40, ms45, ms50, ms55, ms60, ms65, ms70, ms75, ms80, ms85, ms90, ms95, ms100, ms105, ms110, ms115, ms120, ms125, ms130, ms135, ms140, ms145, ms150, ms155, ms160, ms165, ms170, ms175, ms180, ms185, ms190, ms195, ms200, ms205, ms210, ms215, ms220, ms225, ms230, ms235, ms240, ms245, ms250, ms300, ms350, ms400, ms450, ms500 | Prohibit timer for STATUS PDU transmission |

### RLC-UM (Unacknowledged Mode) - Voice Bearer (DRB-5, Voice Call Only)

| Parameter | Observed Value | 3GPP TS 36.331 Range | Purpose |
|-----------|----------------|----------------------|---------|
| **sn-FieldLength (UL)** | size10 | size5, size10 | Sequence number field length (UL) |
| **sn-FieldLength (DL)** | size10 | size5, size10 | Sequence number field length (DL) |
| **t-Reordering (UM)** | Not explicitly set | ms0..ms200 | Timer for UM PDU reordering |

---

## Logical Channel Configuration

### Data Bearers (All Scenarios)

| DRB | LCID | Priority | prioritisedBitRate | bucketSizeDuration | logicalChannelGroup |
|-----|------|----------|--------------------|--------------------|---------------------|
| DRB-3 | 3 | 4 | kBps8 | ms50 | 1 |
| DRB-4 | 4 | 8 | kBps8 | ms50 | 3 |

### Voice Bearer (Voice Call Scenario Only)

| DRB | LCID | Priority | prioritisedBitRate | bucketSizeDuration | logicalChannelGroup |
|-----|------|----------|--------------------|--------------------|---------------------|
| DRB-5 | 5 | 3 | kBps8 | ms50 | 1 |

### 3GPP Spec Ranges for Logical Channel Config

| Parameter | Range (3GPP TS 36.331) |
|-----------|------------------------|
| **priority** | 1..16 (1 = highest priority) |
| **prioritisedBitRate** | kBps0, kBps8, kBps16, kBps32, kBps64, kBps128, kBps256, infinity, kBps512-v1020, kBps1024-v1020, kBps2048-v1020 |
| **bucketSizeDuration** | ms50, ms100, ms150, ms300, ms500, ms1000 |
| **logicalChannelGroup** | 0..3 |

---

## Scenario-Wise Parameter Changes

### Common Configurations (All Scenarios)
All scenarios share the same base MAC and RLC configurations:
- **DRX**: onDurationTimer=psf10, drx-InactivityTimer=psf100, drx-RetransmissionTimer=psf8
- **RLC-AM**: t-PollRetransmit=ms80, t-Reordering=ms35, maxRetxThreshold=t32

### Voice Call - Unique Configurations

| Parameter | Value | Difference from Data-Only |
|-----------|-------|---------------------------|
| **Voice Bearer (DRB-5)** | Present | Not present in other scenarios |
| **RLC Mode for Voice** | UM (Unacknowledged) | Data bearers use AM |
| **Voice Priority** | 3 (highest) | Data is priority 4/8 |
| **ROHC Header Compression** | Enabled | Not used for data |
| **EPS Bearer ID (Voice)** | 7 (QCI 1) | Data uses EPS 5/6 |

### Data-Intensive Scenarios (100MB Download, Browsing)

| Observation | Value |
|-------------|-------|
| **Carrier Aggregation** | Frequent SCell modifications |
| **RRC Activity** | Higher (0.66 msg/sec for 100MB vs 0.24 for voice) |
| **EPS Bearer Swap** | DRB-3 EPS: 6→5, DRB-4 EPS: 5→6 (during SCell activation) |

---

## Key Findings

### 1. MAC Configuration Consistency
The MAC-MainConfig (especially DRX parameters) remains **consistent** across all scenarios. The network uses a standard DRX profile for power saving:
- Short cycle: 80ms
- Long cycle: 320ms
- Inactivity timer: 100ms

### 2. RLC Mode Selection by Service Type

| Service Type | RLC Mode | Rationale |
|--------------|----------|-----------|
| Voice (VoLTE) | UM (Unacknowledged) | Low latency, no retransmissions |
| Data (Browsing/Download) | AM (Acknowledged) | Reliable delivery with ARQ |

### 3. Priority Differentiation

```
Voice Bearer (DRB-5):  Priority 3 (Highest)
Data Bearer (DRB-3):   Priority 4 (Medium)
Data Bearer (DRB-4):   Priority 8 (Lowest)
```

### 4. Parameters That Changed Between Scenarios

| Parameter | Voice Call | Data Sessions | Trigger for Change |
|-----------|------------|---------------|-------------------|
| **DRB-5 (Voice Bearer)** | Present (UM) | Absent | VoLTE call establishment |
| **EPS Bearer IDs** | Initial: DRB3=6, DRB4=5 | Swapped during SCell | Carrier Aggregation |
| **RRC Message Rate** | 0.24 msg/sec | 0.33-0.66 msg/sec | Traffic intensity |

---

## 3GPP Reference Specifications

- **3GPP TS 36.331**: RRC Protocol specification (defines MAC-MainConfig, RLC-Config IEs)
- **3GPP TS 36.321**: MAC Protocol specification (defines DRX behavior)
- **3GPP TS 36.322**: RLC Protocol specification (defines AM/UM modes)
- **3GPP TS 36.323**: PDCP Protocol specification (defines ROHC for VoLTE)

---

## Summary Table: All Extracted Parameters

| Category | Parameter | Value(s) Observed | 3GPP Range |
|----------|-----------|-------------------|------------|
| **MAC-DRX** | onDurationTimer | psf10 | psf1-psf200 |
| **MAC-DRX** | drx-InactivityTimer | psf100 | psf1-psf2560 |
| **MAC-DRX** | drx-RetransmissionTimer | psf8 | psf1-psf33 |
| **MAC-DRX** | longDRX-CycleStartOffset | sf320 | sf10-sf2560 |
| **MAC-DRX** | shortDRX-Cycle | sf80 | sf2-sf640 |
| **RLC-AM** | t-PollRetransmit | ms80 | ms5-ms4000 |
| **RLC-AM** | pollPDU | p16 / pInfinity | p4-pInfinity |
| **RLC-AM** | pollByte | kBinfinity | kB25-kBinfinity |
| **RLC-AM** | maxRetxThreshold | t32 | t1-t32 |
| **RLC-AM** | t-Reordering | ms35 | ms0-ms200 |
| **RLC-AM** | t-StatusProhibit | ms0 / ms25 | ms0-ms500 |
| **RLC-UM** | sn-FieldLength | size10 | size5, size10 |
| **LogCh** | priority | 3, 4, 8 | 1-16 |
| **LogCh** | prioritisedBitRate | kBps8 | kBps0-infinity |
| **LogCh** | bucketSizeDuration | ms50 | ms50-ms1000 |
| **LogCh** | logicalChannelGroup | 1, 3 | 0-3 |
