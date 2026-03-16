# Smart RRC Reconfiguration: Project Status & Findings Presentation

## Slide 1: Title Slide
**Title:** Smart RRC Reconfiguration Dataset: Building the Ground Truth
**Subtitle:** Overcoming MobileInsight Limitations & Correlating Cross-Layer Cellular Traffic
**Speaker:** [Your Name]

---

## Slide 2: The Core Objective
**Title:** The Goal: Predictive RRC Management
**Talking Points:**
*   **The Problem:** Unnecessary RRC Reconfigurations cause latency spikes and degrade user experience (especially for real-time apps like gaming and VoLTE), while delayed reconfigurations waste radio resources.
*   **The Solution:** Build a "Smart RRC Reconfiguration" ML model that predicts when a reconfiguration *should* happen based on lower-layer network pressure.
*   **The Requirement:** To train this model, we need precise, millisecond-level ground truth data correlating high-level user activity (Browsing, Calls) with low-level modem behavior (MAC Buffer Status, RRC Measurement Reports).

---

## Slide 3: The Tooling Challenge - MobileInsight
**Title:** Challenges with MobileInsight v6.0.0
**Talking Points:**
To capture low-level radio data, we relied on the open-source MobileInsight telemetry tool. However, out of the box, it was fundamentally broken for our modern environment (Ubuntu 24.04, Python 3.12) and our newer 5G chipset (Snapdragon 8 Gen 2 / OnePlus 11R).

**Key Issues Encountered:**
1.  **Python 3.12 Incompatibility:** The core `dm_collector_c` C++ library caused hard segmentation faults due to outdated Python API calls (`Py_ssize_t` casting issues).
2.  **Wireshark Dissector Failures:** The traditional way to view MobileInsight logs is exporting them to PCAP for Wireshark. However, their `ws_dissector` integration was completely broken for modern Wireshark versions, preventing cross-layer inspection.
3.  **Missing MAC Support for SnapDragon 8 Gen 2:** The C-decoder threw "Unknown packet version" errors for LTE MAC Uplink/Downlink Transport blocks because Qualcomm updated their diagnostic packet headers (to version `0x30` and `0x32`), which MobileInsight did not recognize.
4.  **Missing PHY Data:** The Qualcomm modem wasn't sending PHY packets (RSRP/RSRQ) by default. The capture script uses standard equipment IDs but lacks the specific, undocumented Qualcomm diagnostic (`0x73`) commands required to enable raw PHY metrics.

---

## Slide 4: Engineering Solutions & Custom Parser
**Title:** Patching the Pipeline & Creating our own Parser
**Talking Points:**
Because the default toolchain was failing, we reverse-engineered and patched the C++ source code. More importantly, we abandoned the Wireshark UI approach entirely to build an ML-ready pipeline.

**Our Fixes:**
1.  **C-Level Memory Fixes:** Patched `dm_collector_c` to safely handle Python 3.12 tuples, eliminating the segmentation faults.
2.  **MAC Version Patching:** Modified `log_packet.cpp` and `consts.h` to recognize the OnePlus 11R's newer MAC packet versions, successfully restoring 100% visibility of MAC grants and buffers.
3.  **Custom Extraction Pipeline (Why we built our own parser):** 
    *   **The "Why":** Wireshark is great for visual inspection by humans, but terrible for training Machine Learning models because extracting correlated cross-layer features from PCAPs is slow and error-prone.
    *   **The "How":** We wrote a custom Python parser (`parse_logs_to_csv.py`) that hooks directly into the MobileInsight analyzer loop (the `OfflineReplayer`), flattening complex ASN.1 XML trees and MAC sub-headers into a chronological CSV.
4.  **Validation Strategy (How we know the parser is right):**
    *   *Core Engine Trust*: Our parser doesn't guess bits. It hooks into MobileInsight's core `dm_collector_c` engine, which uses official **3GPP ASN.1 templates**. We bypassed the broken formatting layer, not the math.
    *   *Protocol Verification*: We mathematically proved the parser's accuracy. For example, when we initiated a Voice Call, our parser perfectly extracted **LCID 8** (the standard QCI-1 dedicated bearer for VoLTE), sandwiched between two NAS ESM bearer-setup messages. The network protocol behavior perfectly matches our parser's CSV output.

---

## Slide 5: The Experiment - Proving the Concept
**Title:** Data-Off vs. Active Data Analysis
**Talking Points:**
To prove our patched pipeline worked, we captured two 60-second diagnostic logs on the live network:
1.  **Data OFF (Idle):** No user activity.
2.  **Data ON (Active):** Browsing and streaming.

**The Volume Difference:**
*   Data OFF: 180 KB raw binary, 597 decoded events.
*   Data ON: 3.6 MB raw binary, 11,772 decoded events (a 20x increase).

This massive volume shift confirmed we were successfully capturing real user-plane dynamics, not just background maintenance.

---

## Slide 6: Key Finding 1 - RRC Behavior
**Title:** Aggressive RRC Management During Data
**Talking Points:**
When data was actively flowing, the network's signaling behavior changed dramatically.

*   **RRC Reconfigurations** jumped from 30 to 92 (a 3x increase).
*   **Measurement Reports** (the phone telling the network about signal quality) skyrocketed from 24 to 191 (an 8x increase).
*   **The Insight:** Measurement Reports are the primary leading indicator for an RRC Reconfiguration. If the model sees a spike in these reports, a reconfiguration is imminent.

---

## Slide 7: Key Finding 2 - MAC Layer Proof
**Title:** Identifying User Traffic Inside the Enigma
**Talking Points:**
How do we mathematically prove what the user was doing just by looking at the modem logs? We look at **Logical Channel IDs (LCIDs)** inside the MAC Transport Blocks.

*   **Data OFF:** Traffic was restricted exclusively to Control Channels (LCID 1 & 2 for RRC/NAS signaling).
*   **Data ON:** The network activated 5 distinct Data Radio Bearers (DRBs: LCIDs 3, 4, 5, 6, 7). Total Uplink traffic leaped from 3.3 MB to 97.4 MB.
*   **3-Minute Mixed Capture Result:** When we introduced a Voice Call, we immediately saw the activation of **LCID 8**, a dedicated QCI-1 bearer for VoLTE, alongside the internet bearers.

---

## Slide 8: Key Finding 3 - The Buffer Secret
**Title:** MAC Buffer Status Reports (BSR)
**Talking Points:**
The most critical feature we discovered for the ML model is the **Short Buffer Status Report (S-BSR)**.
*   In 5G/LTE, the phone cannot just transmit data; it must request permission.
*   We successfully extracted BSR triggers from the MAC headers (e.g., `S-BSR|4|Padding`).
*   **The Insight:** A BSR indicates the phone's internal buffer is filling up. A high frequency of BSRs indicates the current RRC configuration is insufficient for the user's demand. The ML model will use BSR density to predict "Scale-Up" reconfigurations.

---

## Slide 9: Moving Forward - The Integration Strategy
**Title:** Integrating UE Automation with MobileInsight
**Talking Points:**
To train the "Smart RRC" model, we need thousands of labeled examples. We have two powerful but separate systems that we must now merge.

1.  **The UE Automation Framework (`ue_test.sh`):**
    *   *What it does:* Orchestrates 20 different user scenarios (Browsing, YT, VoLTE, Ping Floods) and generates a `labels.csv` with exact start/stop epoch timestamps for every activity.
2.  **The MobileInsight Pipeline (`raw_capture.py`):**
    *   *What it does:* Silently records the deep modem telemetry (MAC grants, RRC reconfigs) with millisecond precision.

---

## Slide 10: The ML Dataset Pipeline (Next Steps)
**Title:** Building the Ultimate Training Set
**Talking Points:**
Here is our exact roadmap for next week:

1.  **Simultaneous Capture:** Update `ue_test.sh` to trigger the MobileInsight logging script in parallel with the Android Logcat capture.
2.  **Dataset Labeling (The Merge):** Write a Python script to cross-reference the epoch timestamps from the UE Automation `labels.csv` with the MobileInsight `capture.csv`.
3.  **The Final Vector:** Every single MAC/RRC packet will now have a column labeling *exactly what the user was doing* (e.g., `Activity: Call+Browse`).
4.  **Feature Extraction:** Convert the flat CSV into 500ms "rolling windows", aggregating BSR counts, UL grant bytes, and Measurement Reports to predict the binary target: *Will an RRC Reconfiguration happen in the next 500ms?*

---
**Slide 11: Q&A**
**Title:** Questions & Discussion
*   "We have the pipeline, the data format, and the ground truth strategy. We are now ready for large-scale data collection and ML model training."
