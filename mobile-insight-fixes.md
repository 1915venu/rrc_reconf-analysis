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
**Title:** Technical Challenges with MobileInsight v6.0.0
**Talking Points:**
To capture low-level radio data, we relied on the open-source MobileInsight telemetry tool. However, out of the box, it was fundamentally broken for our modern environment (Ubuntu 24.04, Python 3.12) and our newer 5G chipset (Snapdragon 8 Gen 2 / OnePlus 11R).

**Deep Technical Issues Encountered:**
1.  **Mobile App vs. Laptop Capture (OS Limitations):** We initially attempted to use the MobileInsight Android `.apk` directly on the device. However, on Android 15, aggressive SELinux policies and restricted access to `/dev/diag` caused the app to throw "unsupported chipset" errors. We pivoted to a **laptop-based approach**, using ADB to route the Qualcomm Diagnostic USB interface (`sys.usb.config diag,adb`) directly to a Linux serial port (`/dev/ttyUSB0`), bypassing the Android OS sandboxing entirely.
2.  **C-Level Segmentation Faults (Python 3.12 API Breaking Change):** The core `dm_collector_c` C++ library caused hard segmentation faults the moment it attempted to parse a packet. Root cause: MobileInsight uses deprecated C-API calls (`PyArg_ParseTupleAndKeywords`) that expect `int` lengths, but Python 3.12 strictly enforces `Py_ssize_t`. This caused massive memory corruption during hardware log ingestion.
3.  **Missing RRC Support for SnapDragon 8 Gen 2:** The analyzer immediately threw "Unknown Component Version `27`" errors for `LTE_RRC_OTA_Packet` messages, completely blinding us to signaling events. Root cause: The analyzer only supported RRC packet header versions up to `17`. The new Qualcomm modem uses a structurally updated version 27 header that the MobileInsight decoder simply didn't know how to parse.
4.  **Missing MAC Support for SnapDragon 8 Gen 2:** Similarly, the C-decoder dropped all LTE MAC packets due to "Unknown packet version" errors. Root cause: The modem shifted MAC Transport Block header versions from `v0x24` to `v0x30` (Uplink) and `v0x32` (Downlink), rendering MobileInsight's internal `switch-case` statements useless.
5.  **Missing PHY Data (Undocumented Diag Commands):** The default capture script enables log masks by "Equipment ID" (which covers RRC/MAC/NAS), but fails to capture PHY layer metrics (RSRP/RSRQ/SINR). Root cause: Qualcomm modems require highly specific, undocumented `0x73` (Log Config) diagnostic command payloads to explicitly turn on individual PHY sub-packet streams (like `0xB193` for Serving Cell Measurements).

---

## Slide 4: Engineering Solutions & Custom Parser
**Title:** Patching the C++ Pipeline & Bypassing Wireshark
**Talking Points:**
Because the default toolchain was failing at the memory, decoding, and processing layers, we reverse-engineered the source code to build a Custom Extraction Pipeline optimized specifically for Machine Learning.

**Our Technical Fixes:**
1.  **C-Level Memory Fixes:** We manually patched the `msg_len` variables in `dm_collector_c` from standard C `int` to `Py_ssize_t` to satisfy Python 3.12's strict C-API memory requirements, immediately resolving all segmentation faults and restoring runtime stability.
2.  **RRC OTA Version Patching:** We discovered that the new RRC version 27 payload structure is identical to version `15`/`16`/`17` after the internal header. We modified `log_packet.cpp` to explicitly recognize headers with component version `27`, restoring our RRC signaling visibility.
3.  **MAC Version Patching & Recompilation:** We similarly modified the core decoding engine for the MAC layer. We added new `case` statements for the `v0x30` and `v0x32` MAC headers and mapped them to the existing payload extraction logic. We then recompiled the `dm_collector_c.so` shared object, successfully restoring 100% visibility of the vital MAC grants and buffer status reports.
4.  **Why We Abandoned Wireshark (The Parser Necessity):** 
    *   **The Wireshark Limitation (RRC vs. MAC mismatch):** We initially built the `ws_dissector` script to dump logs to a PCAP. Wireshark's built-in ASN.1 templates successfully decrypted and visualized the rich **RRC signaling trees**. However, the **MAC payload dissection remained opaque** because MobileInsight's GSMTAP encapsulation mapped the proprietary Qualcomm MAC sub-headers incorrectly, breaking Wireshark's GUI view.
    *   **The "Why" (Data Structural Conflict):** We needed programmatic, structural access to the multiplexed MAC LCIDs (Data Pipes) and BSRs (Buffer Reports) to mathematically correlate them with RRC events. Wireshark is designed for human visual inspection. Attempting to extract deeply nested, multiplexed MAC headers out of a visual PCAP file into a flat Machine-Learning feature vector is computationally slow, lossy, and highly error-prone.
5.  **The Custom Parser Architecture (`parse_logs_to_csv.py`):**
    *   **The "How":** We wrote a Python observer class that hooks directly into the MobileInsight `OfflineReplayer` event loop. It intercepts the decoded dictionaries *before* they are ever formatted for Wireshark or the GUI. It natively traverses the ASN.1 ElementTrees and unpacks the MAC sub-header arrays in pure Python, flattening them into a high-octane, chronologically ordered CSV.
    *   **Validation Strategy (How we know it's mathematically sound):** We bypassed the broken formatting layer, but importantly, we did *not* reinvent the decoding math. We rely entirely on the official 3GPP ASN.1 templates hardcoded into the C++ engine. We proved our parser's accuracy behaviorally: for example, the moment our script detected a Voice Call initiation (via NAS ESM), our parser perfectly extracted the activation of **LCID 8** (the 3GPP standard QCI-1 dedicated bearer for VoLTE). Our extracted data matches the theoretical LTE specifications flawlessly.

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
