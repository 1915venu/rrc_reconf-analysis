# LTE Log Capture & ML Dataset Extraction Guide

This guide details the end-to-end process to replicate the 60-second diagnostic logging sessions on your OnePlus 11R and parse the data into an ML-ready CSV.

---

## 1. Environment & Prerequisites

1. **Activate the Virtual Environment:**
   All scripts must be run inside the MobileInsight virtual environment to ensure dependencies are met.
   ```bash
   source /home/venu/mobileinsight-core/venv/bin/activate
   ```

2. **Fix Path Conflicts (If applicable):**
   If Python complains about importing MobileInsight, ensure your scripts start with:
   ```python
   import sys
   # Prevent loading the local MobileInsight-6.0.0 folder over the installed core
   sys.path = [p for p in sys.path if 'MobileInsight-6.0.0' not in p]
   from mobile_insight.monitor.dm_collector import dm_collector_c
   ```

---

## 2. Enabling Qualcomm Diagnostic Mode

The phone's `/dev/ttyUSB0` diagnostic port must be exposed before capturing.

1. **Connect your phone** via USB with ADB debugging enabled.
2. **Enable Diag Mode** (requires root):
   ```bash
   adb shell su -c "setprop sys.usb.config diag,adb"
   ```
3. **Wait 5 seconds** and verify the port exists:
   ```bash
   ls -l /dev/ttyUSB0
   ```
4. **Grant permissions** to your user to read/write the port:
   ```bash
   sudo chmod 666 /dev/ttyUSB0
   ```

---

## 3. Capturing Raw Logs (.bin)

We use the raw capture script you built earlier to record 1 minute of LTE RRC, MAC, and NAS packets.

**Command:**
```bash
python3 scripts/raw_capture.py /dev/ttyUSB0 115200 60 logs/new_capture.bin
```
* **`/dev/ttyUSB0`**: The diag port.
* **`115200`**: The baud rate.
* **`60`**: Duration in seconds. Can be adjusted.
* **`logs/new_capture.bin`**: Output file containing raw HDLC-framed qualitative packets.

*Note: During this 60 seconds, either leave the phone completely idle (for a control dataset) or actively browse the web/stream video (to capture heavy MAC/RRC reconfiguration traffic).*

---

## 4. Converting Raw Binary to .mi2log

The raw `.bin` file needs to be converted into MobileInsight's internal `.mi2log` format so our offline analyzers and parsers can decode the specific packets.

Because of a path conflict in the workspace, we used a simplified script `/tmp/convert_only.py` to do this reliably:

**Command:**
```bash
python3 /tmp/convert_only.py logs/new_capture.bin logs/new_capture.mi2log
```

---

## 5. Parsing .mi2log to CSV (Feature Extraction)

We take the parsed packets and flatten them into a single chronological CSV using the script you developed earlier. This extracts only the necessary features for ML (RRC message types, MAC grants, buffer status).

**Command:**
```bash
python3 scripts/parse_logs_to_csv.py logs/new_capture.mi2log logs/new_capture.csv
```

### Explanation of the Output Columns:
- **Timestamp**: High precision microsecond timestamp to help correlate MAC throughput with RRC drops.
- **Layer & Message**: `RRC`, `MAC`, or `NAS`.
- **RRC Configuration**: Specific parameters sent by the network (e.g., Measurement configuration, DRB Setup).
- **MAC Statistics**: Identifies active channels (LCIDs) and bytes granted (Transport Block Size).
- **Internal Buffer Status**: Tells you exactly how much data is sitting in the phone's transmit queues (highly predictive of impending network reconfigurations).

---

## 6. Building the Machine Learning Model (Next Steps)

With your `.csv` files ready (like `capture_data_off.csv` and `capture_active_data_1min.csv`), the next step is building the ML dataset:

1. **Create Rolling Windows**: Map the chronological packets into fixed windows (e.g., 500ms).
2. **Aggregate MAC Features**: In each window, sum the UL/DL MAC grant bytes, count the unique active DRBs (LCIDs 3-7), and average the internal buffer statuses.
3. **Extract Label**: Does an `LTE_RRC_OTA_Packet` with `rrcConnectionReconfiguration` occur in the *next* 500ms window? `1` for Yes, `0` for No.
4. **Train**: Feed these windows into a classifier (Random Forest, XGBoost, or an LSTM for sequential prediction). You will find that high buffer queues or an `RRC MeasurementReport` are massive leading indicators for the `1` label.
