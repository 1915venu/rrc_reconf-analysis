# Sigma ML Logs — Field Analysis

## Overview

| Property | Value |
|----------|-------|
| **Source** | QXDM / Sigma diagnostic tool — LTE MAC layer logs |
| **Timestamp** | 2026-02-18 ~13:00 UTC+05:30 |
| **Files** | 4 [.xlsx](file:///home/venu/Downloads/logs_sigma_ml/MACNone_2026.02.18.12.59.51_1_1.xlsx) files, ~65,938 total rows |
| **Total Packet Events** | **407** (236 DL TB + 165 UL TB + 3 RACH Trigger + 3 RACH Attempt) |

---

## File Breakdown

| File | Rows |
|------|------|
| `MACNone_..._1_1.xlsx` | 12,925 |
| `MACNone_..._1_2.xlsx` | 20,095 |
| `MACNone_..._1_3.xlsx` | 20,123 |
| `MACNone_..._1_4.xlsx` | 12,795 |

---

## Data Structure

Each excel contains **TWO types of data**:

1. **Column 0 (hierarchical text)** — Nested `Key = Value` pairs from RACH and packet metadata
2. **Columns 0–28 (tabular rows)** — Structured tables for Transport Block details, **including LCID**

---

## Message Types & All Fields

### 1. `0xB063` — LTE MAC DL Transport Block (236 events)

Contains **3 nested table schemas**:

#### Schema A: Transport Block Header (10 fields)

| Field | Description |
|-------|-------------|
| `TB Size` | Transport Block size in bytes |
| `Num Pad Bytes` | Number of padding bytes |
| `Frame` | System Frame Number (SFN) |
| `SubFrame` | Subframe number (0–9) |
| `Reparse Flag` | Whether the TB was re-parsed |
| `RNTI Type` | Radio Network Temp ID type (**C_RNTI**: 2328, **T_C_RNTI**: 2) |
| `CC ID` | Component Carrier ID |
| `HARQ ID` | Hybrid ARQ process ID |
| `Num MAC Sdu` | Number of MAC SDUs in this TB |
| `MAC Hdr Len` | MAC header length in bytes |

#### Schema B: MAC SDU / LCID Detail (16 fields) 

| Field | Description |
|-------|-------------|
| **`Is MCE`** | MAC Control Element flag |
| **`LCID`** | **Logical Channel ID**  |
| `Sdu Len` | SDU length |
| `SN` | PDCP Sequence Number |
| `DC` | Data/Control indicator |
| `P` | Polling bit |
| `RF` | Re-segmentation flag |
| `E` | Extension bit |
| `FI` | Framing Info |
| `LSF` | Last Segment Flag |
| `SO` | Segment Offset |
| `Num PDCP Grp` | Number of PDCP groups |
| `Num NLOs` | Number of NLOs |
| `Mac CE Payload` | MAC Control Element payload (raw hex) |
| `Li Num` | Length Indicator count |
| `Li Len` | Length Indicator value |

#### LCID Values (DL) — Distribution

| LCID Value | Count | Meaning |
|------------|-------|---------|
| **3** | 2,259 | DRB1 (Dedicated data bearer — user data) |
| **1** | 53 | SRB1 (Signalling Radio Bearer 1 — RRC) |
| **4** | 36 | DRB2 (Second data bearer) |
| **ACT/DEACT** | 44 | SCell Activation/Deactivation MAC CE |
| **Timing Advance** | 14 | Timing Advance Command MAC CE |
| **2** | 5 | SRB2 (Signalling Radio Bearer 2 — NAS) |
| **28** | 2 | Reserved / Padding |
| **0** | 2 | CCCH (Common Control Channel — for initial access) |

> [!IMPORTANT]
> **LCID is present!** It exists in the DL Transport Block's SDU-level table (Schema B), not in column 0's text. The dominant LCID=3 (DRB1 = user data) shows this was primarily a **data transfer session**.

---

### 2. `0xB064` — LTE MAC UL Transport Block (165 events)

#### UL Grant Table (29 fields)

| Field | Description |
|-------|-------------|
| `Sub Id` | Subscription ID |
| `Cell Id` | Serving cell ID |
| `HARQ ID` | HARQ process ID |
| `RNTI Type` | **C-RNTI** (165 events) |
| `Sub-FN` | Sub-frame number |
| `SFN` | System Frame Number |
| **`Grant (bytes)`** | UL grant size from eNodeB |
| `RLC PDUs` | Number of RLC PDUs transmitted |
| **`Padding (bytes)`** | Unused grant bytes |
| **`BSR event`** | Buffer Status Report trigger reason |
| **`BSR trig`** | BSR type actually sent |
| `HDR LEN` | Header length |
| `Mac Hdr + CE` | Raw hex of MAC header + Control Elements |
| **`LC ID`** | **Logical Channel ID (UL direction)**  |
| `LEN` | Length field |
| `BSR LCG 0–3` | Buffer sizes per Logical Channel Group |
| `BSR LCG 0–3 (bytes)` | Buffer sizes in bytes per LCG |
| `PHR Ind` | Power Headroom Report indicator |
| `Pcmax_c` | Max configured transmit power |
| `PH SCell1/2` | Power Headroom for SCells |
| `Pcmax_c SCell1/2` | Max power for SCells |

#### LC ID Values (UL) — Distribution

| LC ID Value | Count | Meaning |
|-------------|-------|---------|
| **S-BSR** | 79 | Short Buffer Status Report |
| **L-BSR** | 81 | Long Buffer Status Report |
| **3** | 3 | DRB1 (actual user data UL) |
| **Padding** | 1 | Padding LCID |
| **E-PHR** | 1 | Extended Power Headroom Report |

#### BSR Event Distribution

| BSR Event | Count |
|-----------|-------|
| Periodic | 95 |
| High data arrival | 63 |
| Robustness BSR | 1 |

#### BSR Trigger Distribution

| BSR Trigger | Count |
|-------------|-------|
| Pad L-BSR | 81 |
| S-BSR | 79 |
| No BSR | 3 |
| L-BSR | 1 |
| Cancelled | 1 |

---

### 3. `0xB061` — LTE MAC RACH Trigger (3 events)

Hierarchical fields in column 0:

| Field | Sample Values |
|-------|---------------|
| `Rach reason` | Connection Request (2×), Handover (1×) |
| `RACH Contention` | Contention-based (2×), Contention-free (1×) |
| `Preamble` | Preamble index used |
| `Preamble Format` | Format type |
| `Preamble initaial Power` | Initial Tx power |
| `Power ramping step` | Power increase per attempt |
| `PMax` | Max allowed power |
| `Group chosen` | Group A/B |
| `Msg3 size` | Size of Msg3 |
| `Contention Resolution timer` | Timer value |
| `Max retx Msg3` | Max retransmissions |
| `Root Seq index` | ZC root sequence |
| `PRACH Config` | PRACH configuration index |
| `Cell Id`, `Sub Id`, `CRNTI`, `Machine ID` | Identifiers |

---

### 4. `0xB062` — LTE MAC RACH Attempt (3 events)

| Field | Sample Values |
|-------|---------------|
| `Rach result` | **Success** (3×) |
| `Result` | True (3×) |
| `TA value` | Timing Advance from eNB |
| `TCRNTI` | Temporary C-RNTI assigned |
| `Retx counter` | Number of retries |
| `Backoff Value` | Network backoff |
| `Grant` / `Grant Raw` | UL resource grant |
| `Contention procedure` | Procedure type |
| `Preamble Index` | Used preamble |
| `Harq ID` | HARQ process |

---

## Key Observations

1. **LCID is fully present** in both DL (as `LCID`) and UL (as `LC ID`) Transport Blocks
2. **LCID=3 dominates** (2,259 out of 2,415 DL SDUs = **93.5%**) → mostly user-plane data on DRB1
3. **MAC Control Elements** are visible: Timing Advance (14×), SCell ACT/DEACT (44×), BSR, PHR
4. **3 successful RACH events**: 2 Connection Requests + 1 Handover, all successful
5. **BSR activity is heavy**: 160 BSR events, split between Periodic (60%) and High data arrival (39%)
6. **No failures**: All RACH attempts succeeded, no retransmissions observed
