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

---

## 🗄️ File Manifest (What's in every folder)

Inside every `Phase_*` folder, you will find a standardized "Evidence Pack" for the model:

| File Name | Purpose | Description |
| :--- | :--- | :--- |
| **`*.csv.gz`** | **The Feature Matrix** | The core ML input. Contains 31 columns of strictly aligned MAC layer grants, BSR/PHR triggers, and RRC Reconfigurations. |
| **`labels.csv`** | **The Ground Truth** | Accurate timestamp pairs mapping exactly when an activity started and stopped. Used for supervised training. |
| **`execution.log`** | **The Audit Trail** | A human-readable step-by-step trace of what the automation framework executed on the phone. |
| **`session_meta.txt`** | **Random Sequence (Phase C)** | A comma-separated list defining exactly which activities were randomly chosen and chained in that specific batch. |
| **`upload*.log`** | **Background Context** | Text logs capturing the `ping` outputs from background stress tests. |

## 🧬 Key ML Features (The 31 Columns)

The CSVs extract data into perfectly flat timestamps for Time-Series forecasting. Key features include:

*   **L2 Traffic Features:** `Grant_Bytes`, `Has_SBSR`/`Has_LBSR`, `Has_PHR`, `Num_Data_LCIDs`.
*   **L3 State Features:** Binary triggers for critical events like `Is_RRC_Reconf`, `Target_MeasConfig`, `Target_Handover`, `Target_DRB_Setup`.
*   **RRC Detail:** A concatenated string defining the specific IE changes (e.g., `MeasConfig|Handover|TargetPCI=84`).
