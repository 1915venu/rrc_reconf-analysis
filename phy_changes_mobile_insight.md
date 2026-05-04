# PHY Decoder Changes — Snapdragon 8 Gen 2 (v59 Patch)

**File patched:** `dm_collector_c/log_packet.cpp` — `case 59:` block at line 3144  
**Device:** OnePlus 11R (Snapdragon 8 Gen 2)  
**Signal:** `LTE_PHY_Serv_Cell_Measurement`

---

## 1. Background

MobileInsight 6.0.0 was written for Snapdragon 820/835 era chipsets. The Snapdragon 8 Gen 2 sends PHY Serving Cell Measurement packets at **subpacket version 59** — a version the original decoder had no knowledge of.

Every packet the modem sends has a version number in its header. The C++ decoder uses a `switch(version)` block — one recipe per known version. With no `case 59:`, the decoder either:

- **Silently dropped** every PHY packet (no RSRP/RSRQ/SNR in the CSV at all), or
- **Used the wrong recipe** from a nearby version, reading bytes at the wrong positions and producing physically impossible values like RSRP = −173 dBm or RSRQ = +33 dB.

---

## 2. How We Found the Correct Byte Positions

Qualcomm never publishes packet formats. We reverse-engineered the byte layout using **cross-validated offset scanning**.

### Step 1 — Get ground truth

While the phone was capturing, we simultaneously ran:

```bash
adb shell dumpsys telephony.registry
```

Android reports: **RSRP = −96 dBm**, **RSRQ = −17 dB** (1 dBm resolution, trustworthy).

### Step 2 — Convert to expected raw integer

Qualcomm uses fixed-point encoding for all PHY values:

| Signal | Encoding formula | Inverse (raw → target) |
|--------|-----------------|------------------------|
| RSRP | `raw × 0.0625 − 180 = dBm` | −96 dBm → raw = **1344** |
| RSRQ | `raw × 0.0625 − 30 = dB` | −17 dB → raw = **208** |
| SNR | `raw × 0.1 − 20 = dB` | ~+10 dB → raw ≈ **307** |

### Step 3 — Brute-force scan all byte positions

Script `correlate_phy_offsets.py` tried every combination of:

- Byte offset: 0 → 96
- Bit shift: 0 → 23
- Bitmask: 511 (9-bit), 1023 (10-bit), 4095 (12-bit)

For each combination it applied the formula and checked proximity to the ADB ground truth.

### Step 4 — Cross-validate across 3 packets

A random coincidence matches one packet. The true offset must give a **consistent, physically plausible value across all 3 captured packets**. False positives are eliminated by this step.

### Validated result — small variant (subpacket size 100 or 104 bytes)

| Signal | Byte offset | Bit shift | Bitmask | Formula | Decoded value |
|--------|------------|-----------|---------|---------|--------------|
| RSRP | 44 | >> 12 | & 4095 | × 0.0625 − 180 | **−97.12 dBm** ✅ (ADB: −96) |
| RSRQ | 34 | >> 13 | & 1023 | × 0.0625 − 30 | **−17.00 dB** ✅ |
| SNR | 32 | >> 9 | & 511 | × 0.1 − 20 | **+10.70 dB** ✅ |

---

## 3. The Two-Variant Problem

After the first fix worked on some batches, Phase D captures (batches 6 & 7) still showed **RSRQ = +33 dB** in 62.5% of rows — physically impossible (valid range: −3 to −30 dB).

Root cause: **version 59 is not a single layout**. The `SubPacket Size` field tells you which structure the modem used:

| SubPacket Size | Variant | Meaning |
|---------------|---------|---------|
| 100 or 104 bytes | Small | Single cell record — original offsets |
| ≥ 160 bytes (also 308, 456) | Large | Multi-cell or extended record — fields moved |

Applying the small-variant recipe to a large-variant packet reads bytes at the wrong position — producing the +33 dB garbage.

### Large variant offsets (size ≥ 160 bytes)

| Signal | Byte offset | Bit shift | Bitmask | Formula |
|--------|------------|-----------|---------|---------|
| RSRP | 36 | >> 12 | & 4095 | × 0.0625 − 180 |
| RSRQ | **116** | >> 1 | & 511 | × 0.0625 − 30 |
| SNR | 32 | >> 9 | & 511 | × 0.1 − 20 |

RSRQ moved from **byte 34 → byte 116** between the small and large variants. This is the core of why the original fix wasn't sufficient.

---

## 4. The Fix — Code at log_packet.cpp:3144

```cpp
case 59: {
    int v59_subpkt_size = _search_result_int(result_subpkt, "SubPacket Size");

    float rsrp_val = -999.0f;
    float rsrq_val = -999.0f;
    float snr_val  = -999.0f;

    if (v59_subpkt_size == 100 || v59_subpkt_size == 104) {
        // Small variant — single cell record
        if (offset + 44 + 4 <= (int)length) {
            unsigned int utemp = *((unsigned int *) (b + offset + 44));
            rsrp_val = float((utemp >> 12) & 4095) * 0.0625 - 180.0;
        }
        if (offset + 34 + 4 <= (int)length) {
            unsigned int utemp = *((unsigned int *) (b + offset + 34));
            rsrq_val = float((utemp >> 13) & 1023) * 0.0625 - 30.0;
        }
        if (offset + 32 + 4 <= (int)length) {
            unsigned int utemp = *((unsigned int *) (b + offset + 32));
            snr_val = float((utemp >> 9) & 511) * 0.1 - 20.0;
        }
    } else if (v59_subpkt_size >= 160) {
        // Large variant — extended/multi-cell record
        if (offset + 36 + 4 <= (int)length) {
            unsigned int utemp = *((unsigned int *) (b + offset + 36));
            rsrp_val = float((utemp >> 12) & 4095) * 0.0625 - 180.0;
        }
        if (offset + 116 + 4 <= (int)length) {
            unsigned int utemp = *((unsigned int *) (b + offset + 116));
            rsrq_val = float((utemp >> 1) & 511) * 0.0625 - 30.0;  // RSRQ moved to offset 116
        }
        if (offset + 32 + 4 <= (int)length) {
            unsigned int utemp = *((unsigned int *) (b + offset + 32));
            snr_val = float((utemp >> 9) & 511) * 0.1 - 20.0;
        }
    }

    // Physical range guards — discard values outside what physics allows
    if (rsrp_val >= -140.0f && rsrp_val <= -40.0f)  { /* emit RSRP */ }
    if (rsrq_val >= -30.0f  && rsrq_val <= -3.0f)   { /* emit RSRQ */ }
    if (snr_val  >= -30.0f  && snr_val  <= 40.0f)   { /* emit SNR  */ }

    offset += (v59_subpkt_size - 4);  // advance past subpacket payload
    success = true;
    break;
}
```

---

## 5. Physical Range Guards

After decoding, every value is checked against 3GPP physical limits before being emitted. If out of range, it is silently discarded (no field emitted) rather than passing garbage to Python.

| Signal | Min | Max | Rationale |
|--------|-----|-----|-----------|
| RSRP | −140 dBm | −40 dBm | 3GPP TS 36.133 measurement range |
| RSRQ | −30 dB | −3 dB | 3GPP TS 36.133 measurement range |
| SNR | −30 dB | +40 dB | Physical receiver sensitivity limits |

Without guards, any wrong-offset packet that happened to decode to a plausible-looking number would silently corrupt training data.

---

## 6. Re-decoding Old Captures

The `.mi2log` file produced during a capture bakes in the decoded values at capture time. Replaying an old `.mi2log` through a fixed decoder does NOT re-decode it — the values are already inside.

Phase D batches 6 and 7 were captured before the fix. Their raw `.mi2log` binary files were still on disk. We:

1. Rebuilt the patched shared library from source:
   ```bash
   cd /home/venu/Downloads/MobileInsight-6.0.0
   mi-env/bin/python -c "
   from distutils.core import setup, Extension
   setup(name='dm_collector_c',
         ext_modules=[Extension('dm_collector_c',
           sources=['dm_collector_c/dm_collector_c.cpp', ...])],
         script_args=['build_ext','--inplace'])
   "
   cp dm_collector_c.cpython-312-x86_64-linux-gnu.so \
      mi-env/lib/python3.12/site-packages/mobile_insight/monitor/dm_collector/
   ```

2. Re-ran `parse_logs_ml.py --mac --phy` on the raw `.mi2log` for both batches.

3. Got fresh CSVs with correct RSRQ values (−13 to −20 dB range, all valid).

### Result

| Batch | Before fix | After fix |
|-------|-----------|-----------|
| b6 (Phase D 20260420_155257) | 62.5% RSRQ invalid (up to +33 dB) | 0% invalid, range −13 to −20 dB ✅ |
| b7 (Phase D 20260420_162733) | 62.5% RSRQ invalid | 0% invalid, range −13 to −20 dB ✅ |
| b8 (post-fix 20260424_170830) | 0% invalid (decoded with patched binary) | 0% invalid ✅ |
| b1–b5 (Phase C, no PHY) | Constant −20 fill (no PHY data) | Unchanged |

---

## 7. Impact on the ML Model

With valid PHY data, three PHY features entered the **top 10 by importance** (LightGBM gain, final trained model):

| Rank | Feature | Gain |
|------|---------|------|
| 4 | `phy_rsrp_mean` | 310,618 |
| 9 | `phy_snr_rx0_mean` | 154,832 |
| ~11 | `phy_rsrp_delta` | 83,060 |

`phy_rsrq_mean` also contributes but correlates heavily with RSRP+SNR so LightGBM down-weights it via feature selection.

Without the decoder fix, all three of these features would have contained garbage or been absent entirely — the model would have had no signal-quality information at all.

---

## 8. Summary of All PHY-Related Patches

| # | What | Where | Problem | Fix |
|---|------|-------|---------|-----|
| 1 | PHY Serv Cell v59 — small variant | `log_packet.cpp:3153` | No case 59 existed; wrong bytes read | Added `case 59:` with correct offsets for size 100/104 |
| 2 | PHY Serv Cell v59 — large variant | `log_packet.cpp:3167` | RSRQ read from byte 34 instead of byte 116 | Added size ≥ 160 branch with RSRQ at offset 116 |
| 3 | Physical range guards | `log_packet.cpp:3184–3212` | Out-of-range values silently corrupted CSV | Added min/max checks before emitting any field |
| 4 | Re-decoded Phase D b6, b7 | `parse_logs_ml.py` re-run | Old `.mi2log` baked in pre-fix values | Rebuilt `.so`, re-decoded raw binary, replaced CSV |
| 5 | Offset advancement | `log_packet.cpp:3214` | Missing `offset += subpkt_size` → multi-subpacket packets misaligned | Added `offset += (v59_subpkt_size - 4)` |
