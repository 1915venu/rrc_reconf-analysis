# 5G RRC Predictive ML Dataset

This directory contains high-fidelity 5G MAC and RRC layer telemetry data captured directly from a Qualcomm modem using MobileInsight. It is explicitly designed for training a Smart RRC Predictor algorithm.

## 📂 Data Hierarchy (The 3 Phases)

The data collection campaign is split into three distinct phases to ensure the Machine Learning model learns both pure signaling signatures and real-world combinatorial chaos.

### 1. Phase A: Pure Signatures (Baselines)
**Goal:** Teach the model what individual activities look like in a vacuum.
*   `Phase_A_Batch_1_VoiceCall`: Exact signaling for VoLTE dedicated bearer setup (QCI=1).
*   `Phase_A_Batch_2_Browsing`: The bursty, intermittent downlink pattern of reading web pages.
*   `Phase_A_Batch_3_DataToggle`: Pure network attach/detach procedures.
*   `Phase_A_Batch_4_UploadStress`: Continuous, saturated Uplink (1400-byte ping flood).

### 2. Phase B: Gap Profiling
**Goal:** Profiling the Idle-to-Active transition sequence.
*   Contains 4 batches focused exclusively on the modem waking up from deep sleep (`RRC_IDLE`).
*   Captures the exact sequence of NAS Attach → RRC Setup → Default Bearer establishment.

### 3. Phase C: The Chaos Matrix
**Goal:** Stress-testing the model with unpredictable, real-world combinations.
*   Contains **5 heavily randomized batches** (e.g., `Phase_C_Batch_1_Chaos...`).
*   In these files, activities overlap (e.g., streaming YouTube *while* receiving a phone call *while* a background upload occurs).
*   This teaches the model not to just memorize scripts, but to actually understand complex feature profiles.

### 4. Phase D: The PHY Breakthrough (49 Columns)
**Goal:** Integrating reverse-engineered Qualcomm physical layer tensors.
*   Unlike Phases A/B/C (31 columns), Phase D includes **49 columns**.
*   We successfully cracked the Snapdragon 8 Gen 2 proprietary `0x3B` Physical layer packet format.
*   Phase D logs natively include `PHY_RSRP`, `PHY_RSRQ`, and `PHY_SNR`, capturing up to 100,000 continuous RF physical samples per batch to predict `MobilityControlInfo` (Handovers).

---

## 🗄️ File Manifest (What's in every folder)

Inside every `Phase_*` folder, you will find a standardized "Evidence Pack" for the model:

| File Name | Purpose | Description |
| :--- | :--- | :--- |
| **`session_random.csv.gz`** | **The Feature Matrix** | The core ML input. Contains 31 columns of strictly aligned MAC layer grants, BSR/PHR triggers, and RRC Reconfigurations. |
| **`labels.csv`** | **The Ground Truth** | Accurate timestamp pairs mapping exactly when an activity started and stopped (e.g., `VoiceCall`, `START`, `16:23:58`). |
| **`execution.log`** | **The Audit Trail** | A human-readable step-by-step trace of the automation framework (e.g., "Dialing 9310...", "Opening Chrome"). |
| **`session_meta.txt`** | **Random Sequence** | A comma-separated list defining exactly which activities were randomly chosen and in what sequence for this specific batch. |
| **`session_random.log`** | **Raw MobileInsight Log** | The raw text-decoded version of the Diag logs containing every single packet before it was filtered into the CSV matrix. |
| **`session_random.mi2log.capture.log`** | **Diagnostic Output** | The standard output and status dump from the internal MobileInsight Python parser. |
| **`radio_props_post_random_session.txt`** | **Hardware State** | A snapshot of the modem's physical properties (RSRP, RSRQ, band, cell ID) taken immediately after the session finishes. |
| **`telephony_state_post_random_session.txt`** | **Network State** | A complete dump of the Android telephony registry (`dumpsys telephony.registry`), showing data connection states and active PLMN. |
| **`upload_stress.log`** | **Foreground Payload** | Text log capturing the explicit `ping` output from a foreground Upload Stress scenario. |
| **`upload_bg_full.log`** | **Background Payload 1** | Text log capturing the output of the background ping flood during a Full Stress scenario (Call+Upload+Download). |
| **`upload_bg_dl.log`** | **Background Payload 2** | Text log capturing the output of the background ping flood during a simultaneous Download scenario. |

## 🧬 Key ML Features (The Columns)

The CSVs extract data into perfectly flat timestamps for Time-Series forecasting. Key features include:

*   **L1 Physical Features:** (Phase D ONLY) `PHY_RSRP`, `PHY_RSRQ`, `PHY_SNR`, `PHY_Rank`, `PHY_CQI`.
*   **L2 Traffic Features:** `Grant_Bytes`, `Has_SBSR`/`Has_LBSR`, `Has_PHR`, `Num_Data_LCIDs`.
*   **L3 State Features:** Binary triggers for critical events like `Is_RRC_Reconf`, `Target_MeasConfig`, `Target_Handover`, `Target_DRB_Setup`.
*   **RRC Detail:** A concatenated string defining the specific IE changes (e.g., `MeasConfig|Handover|TargetPCI=84`).
