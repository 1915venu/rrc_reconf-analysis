# Project Explanation Script — Professor Meeting

**Date:** 2026-05-06
**Project:** Smart RRC Reconfiguration Predictor — End-to-End Status

> Read this top to bottom. Each section is self-contained and roughly 1-2 minutes spoken aloud.
> Bold lines are key claims. Italics are *what to say if asked deeper*.

---

## Part 1 — The Goal (60 seconds)

> *Start here. Sets the scene.*

**The objective is to predict RRC Reconfiguration events on a live 4G LTE phone, before they actually happen — with about 500 milliseconds of lead time.**

RRC Reconfiguration is the message the base station (eNB) sends to a phone to change its radio configuration — adding a data bearer, switching to a different cell, updating measurement settings, activating carrier aggregation, and so on. These are the most important control-plane events in LTE.

If we can predict them ahead of time, three real applications become possible:
1. **Proactive handover preparation** — the phone could pre-load neighbor cell parameters before the handover command arrives, reducing the interruption gap.
2. **Battery optimization** — the modem could stay in deep sleep longer, only waking up when a reconfig is genuinely imminent instead of every DRX cycle.
3. **QoS-aware resource scheduling** — pre-allocate buffers before the throughput spike that follows secondary-cell activation.

I started with **DRB Setup** as the primary target because it's the most signal-rich subtype and has clear cross-layer precursors. The pipeline supports all 11 reconfig types — switching targets is a one-line code change.

---

## Part 2 — The Data Source Problem (90 seconds)

> *This explains why the project required so much low-level work upfront.*

**The first challenge was getting useful data at all.**

Standard Android APIs only expose RSRP and RSRQ at one-second granularity. They don't expose MAC scheduling decisions, HARQ retransmissions, buffer status reports, or any of the per-subframe events that actually drive reconfig decisions. Without those, prediction is impossible.

To get this data, I had to access the **Qualcomm Diagnostic Monitor port** — a proprietary debug interface inside the modem chip. The path is:

1. Phone is rooted (OnePlus 11R, Snapdragon 8 Gen 2).
2. On the phone, set `sys.usb.config diag,adb` — this exposes the diag port over USB.
3. On the laptop, that port appears as `/dev/ttyUSB0` at 115200 baud.
4. **MobileInsight 6.0.0** is an open-source tool that knows how to speak the Qualcomm DM protocol.

This gives me everything: every RRC message, every MAC transport block with grant size and BSR flags, every PHY measurement at ~25 ms granularity, every NAS event.

**But MobileInsight has been broken since 2021** for newer chipsets. That's where most of the engineering effort went.

---

## Part 3 — The C++ Decoder Patches (3 minutes)

> *This is the most impressive technical part. Take your time here.*

MobileInsight was designed for Snapdragon 820 and 835 — chipsets from 2016-2018. The Snapdragon 8 Gen 2 is from 2022. Qualcomm sends slightly different binary packet formats with each chipset generation — same packet types, but with new version numbers and shifted field positions inside the bytes.

When MobileInsight's C++ decoder sees a version number it doesn't know, it either:
- Silently drops the packet (no data at all), or
- Uses an old recipe that reads bytes from the wrong positions, producing garbage like RSRP = −173 dBm or RSRQ = +33 dB. These values are physically impossible but they get emitted and would silently corrupt any ML model.

**I fixed five patches across two C++ files.**

### Patches 1-4 — straightforward fallthroughs

These were cases where the new firmware just incremented the version number but kept the field layout compatible:

| Patch | What it recovered |
|-------|------------------|
| **RRC OTA v27** | All RRC messages — reconfigurations, measurement reports, paging |
| **MAC UL/DL v0x30, v0x32** | Grant bytes, BSR flags, HARQ IDs, LCIDs |
| **PHY Intra-Freq v0x38** | Neighbor cell RSRP — needed for handover precursor signal |
| **PHY PUSCH CSF v0xa3** | CQI (Channel Quality Indicator) and Rank Index |

For these, the fix is a one-line `case` statement falling through to the old decoder. Easy.

### Patch 5 — the hard one: PHY Serving Cell version 59

This packet carries **RSRP, RSRQ, and SNR** — the three fundamental signal quality metrics. On Snapdragon 8 Gen 2 these arrive at version 59, and Qualcomm actually moved the fields to different byte positions. **No legacy fallthrough worked.** I had to reverse-engineer the binary format from scratch.

**The reverse engineering method was:**

1. **Get ground truth from Android.** While capturing, I ran `adb shell dumpsys telephony.registry` simultaneously. Android reports RSRP = −96 dBm, RSRQ = −17 dB. That's my known target.

2. **Convert to expected raw integers.** Qualcomm uses fixed-point encoding: RSRP = `raw × 0.0625 − 180`. So for −96 dBm, the raw integer should be 1344. For RSRQ −17 dB, it should be 208.

3. **Brute-force scan every byte position.** I wrote a Python scanner that tried every byte offset from 0 to 96, every bit shift from 0 to 23, every reasonable bitmask — and checked which combination produced the target value.

4. **Cross-validate across 3 packets.** A coincidence matches one packet. The true offset gives consistent values across all three. That's how I eliminated false positives.

**The validated offsets were:** RSRP at byte 44 with shift 12 mask 4095. RSRQ at byte 34 with shift 13 mask 1023. SNR at byte 32 with shift 9 mask 511. Decoded values matched ADB to within 1.2 dBm.

**Then I hit a second problem.** Some captures still showed RSRQ = +33 dB. It turned out version 59 has **two different internal layouts** depending on the SubPacket Size field — single-cell records use one layout, multi-cell records use another. **RSRQ moved from byte 34 to byte 116** between them.

So the final fix branches on subpacket size, applies different offsets per variant, and includes physical range guards — if a value comes out outside the 3GPP-defined range, it's discarded rather than emitted as garbage.

### Why this matters for ML

**Three of the four PHY features ended up in the top 6 of feature importance.** If the decoder had stayed broken, the model would have had no usable signal-quality features at all. The whole point of cross-layer prediction is to combine MAC traffic patterns with PHY signal patterns — getting PHY working was non-negotiable.

---

## Part 4 — The UE Automation Pipeline (90 seconds)

> *This explains why I have so many batches of data.*

I needed to capture diverse, labelled data systematically. Manual capture isn't reproducible. So I built an automation framework.

**The structure has three layers:**

### Layer 1 — `ue_test.sh` (the per-session runner)

Each session is a randomized sequence of 2 to 20 activities, drawn from a pool of 14 scenarios:
- **High-frequency pool** (more reconfigs per minute): browsing, ping flood, data toggle, idle, traffic burst, voice call. These get 2× weight.
- **Low-frequency pool** (heavier data, fewer reconfigs): YouTube, downloads, uploads, combined activities. Lower weight to save mobile data.

Each activity runs for 45-90 seconds with a 7-second silence "guard band" between activities. The guard band is critical for ML labelling — it gives the model a clean signal for "between activities" so transitions are unambiguous.

Before each session: airplane mode toggle to force a fresh attach. This guarantees the modem starts from a known clean state.

### Layer 2 — `data_campaign.sh` (the multi-batch orchestrator)

Runs N batches in sequence with a 20-second cooldown between them. After each batch:
- Confirms a CSV was generated (parsed from the .mi2log binary).
- If parsing failed for any reason, attempts a recovery parse on the raw binary instead of silently discarding it.
- Once CSV is confirmed, deletes the raw .bin/.mi2log to save disk and compresses the CSV.

This is hardened — there was a bug earlier where a batch timeout could kill the parse step and silently delete the data. Fixed now: data survives all failure modes.

### Layer 3 — `parse_logs_ml.py` (post-capture parser)

Takes a `.mi2log` binary file and produces a 49-column CSV. Each row is one event with timestamp, layer, message type, and ~40 feature columns covering MAC, RRC, NAS, and PHY signals plus 11 reconfig target labels. **All 11 reconfig types are labelled simultaneously** — so when I move from DRB to MeasConfig prediction later, the data is already there.

---

## Part 5 — The Dataset (60 seconds)

> *Quick numbers.*

**Current state: 12 batches, 1,608 DRB Setup positive events, 227,000 windows.**

The data is split across three phases:
- **Phase C (early)** — 5 batches with no PHY data because the v59 decoder wasn't fixed yet
- **Phase D and forward** — 7 batches with full PHY data after the decoder fix

The two earliest Phase D batches were originally captured before the second patch (the SubPacket Size variant fix). After the fix I rebuilt the .so and **re-decoded those binary captures** — recovered them without re-collecting. That's a non-trivial save of work.

Activity distribution among DRB events:
- VoiceCall — 28%
- UploadBG — 15%
- TrafficBurst — 13%
- These three drive 56% of all DRB events. Consistent with telecom expectations: bearer setups happen on call signalling and short data spikes.

**Collection rate: 2-4 batches per day, constrained by mobile data quota.** At this pace I'll reach 3,000 positives in about 2-3 weeks — that's the threshold where model performance should improve substantially.

---

## Part 6 — The ML Pipeline (2 minutes)

### The windowing problem and solution

**Why not just predict on each row?** Because the modem sleeps 97% of the time during DRX. A 100 ms window sitting inside a sleep period contains zero events. Useless.

**My solution: 250 ms sliding windows with 50 ms stride.**
- 250 ms guarantees at least one short DRX cycle (80 ms) is covered, so the window always contains at least one MAC event.
- 50 ms stride means 80% overlap between consecutive windows — 20 windows per second.
- Each window aggregates events into 29 features.

**The label is forward-looking:** "Will a DRB Setup event occur in the next 500 ms after this window ends?" That gives 500 ms of lead time for a downstream system to react.

### The features (29 total)

**MAC features (11):** event count, total/mean/max grant bytes, grant variance, BSR counts (short and long), Power Headroom Report count, padding ratio, unique HARQ IDs in the window, max number of data LCIDs.

**Rate-of-change features (2):** how grant bytes changed vs the previous window, how BSR rate changed. **These are crucial** — the eNB reacts to changes in demand, not absolute levels. A jump from 500 to 5000 bytes per window means a download just started.

**RRC features (3):** measurement report count, whether a reconfig occurred in the window, time since the last reconfig.

**NAS features (1):** ESM events in window — bearer activate/deactivate signals from the network.

**DRX feature (1):** time since the last MAC event — encodes whether the modem is in sleep state.

**PHY features (5):** mean RSRP, RSRQ, SNR, CQI, and RSRP delta vs previous window. **All four are in the top 9 by importance.**

**Activity context (1 + dummies):** which activity is currently happening (one-hot encoded), and how far into that activity we are (capped at 30 seconds to prevent leakage).

### The model

**LightGBM binary classifier.** Why LightGBM and not a neural network?

- **Sample efficiency.** With 1,608 positive examples, gradient boosting outperforms neural networks. Neural networks typically need 10,000+ examples to generalize.
- **Speed.** Trains in 30 seconds, runs inference in under 1 millisecond on ARM. Deployable on the phone itself.
- **Interpretability.** Gives feature importance directly — I can see which signals the model actually uses.

I handle the 1:147 class imbalance with `scale_pos_weight` (effectively up-weighting positive examples). I do NOT use SMOTE — interpolating between time-series points creates synthetic samples that never existed in reality and hurt generalization.

### Evaluation methodology

**Three layers of evaluation:**

1. **Train/Val/Test split** by batch — train on 9 batches, validate on batch 4, test on batches 5+6. Test batches were never touched during development.

2. **Leave-One-Batch-Out cross-validation** — train on 11 batches, evaluate on the 12th, repeat for all 12. This gives a mean AUC-PR with standard deviation showing how stable the model is across sessions.

3. **AUC-PR not AUC-ROC** — with 1:147 imbalance, AUC-ROC is misleading. A model that always predicts "no reconfig" gets a high AUC-ROC because most windows are indeed negative. AUC-PR directly measures precision-recall trade-off, which is what actually matters.

---

## Part 7 — Current Results (60 seconds)

> *Be honest here. The numbers are modest but headed in the right direction.*

**Latest metrics (12 batches):**
- Validation AUC-PR: **0.122**
- Test AUC-PR: 0.070
- LOBO mean AUC-PR: **0.102 ± 0.051**

**The trajectory is what matters:**
- 8 batches → Val 0.086
- 10 batches → Val 0.086 (flat)
- **12 batches → Val 0.122 (+42%)**

This is the first clear signal that adding more data is improving generalization. Up until now the model was overfitting to batch-specific patterns and val performance was stuck flat. Now it's moving.

**Top 6 features by gain:**
1. `time_into_activity_ms` (activity context)
2. `unique_harq_ids` (MAC — bursty traffic indicator)
3. `time_since_last_reconf_ms` (RRC — events cluster temporally)
4. `phy_snr_rx0_mean` (PHY)
5. `phy_rsrp_mean` (PHY)
6. `phy_rsrq_mean` (PHY)

**The importance is healthy.** No single feature dominates. The signal is spread across MAC, RRC, PHY, and NAS layers — exactly what cross-layer prediction should look like. If the decoder fix had failed, none of the PHY features would be there. They are.

---

## Part 8 — Honest Assessment & Limitations (60 seconds)

> *Always include this. Professors respect honesty about limits.*

**What's working:**
- End-to-end pipeline is reproducible. One command rebuilds dataset and retrains.
- Decoder is validated against Android ground truth.
- All 11 reconfig targets are labelled — switching targets is trivial.
- Evaluation is honest (LOBO, AUC-PR, no leakage).

**What's limited:**
- Performance is modest because dataset is still small. Need 3,000+ positives to reach 0.20-0.30 AUC-PR.
- Phone is USB-tethered to the laptop. No mobility data. **Handover prediction won't work** without movement.
- Single device, single operator (Airtel), single network (4G LTE). The model learns this specific eNB's algorithm. Won't transfer to other networks without retraining.

**What's out of scope:**
- ENDC (5G NSA) — Airtel doesn't run NSA on this device. Can't predict events that don't exist in the data.
- SCell_Config — the operator rarely activates carrier aggregation. Too few examples.

---

## Part 9 — Roadmap (30 seconds)

**Next 2-3 weeks:** continue collecting at 2-4 batches per day. Reach 30 batches, ~3,000 positives.

**After that:** train multi-target classifiers — DRB_Setup, MeasConfig, DRB_Release, PHY_Config, MAC_Config — separately. Each gets its own model. Report combined performance table.

**Then:** thesis writeup. Methodology, results, limitations, future work.

**Future work (post-thesis):**
- Battery-powered captures during walking → enable handover prediction
- Different operators (Jio) → test cross-network transfer learning
- A 5G-NSA-capable device → enable ENDC prediction

---

## Part 10 — Quick Reference for Q&A

> *Have these numbers ready in case the prof asks specifics.*

| Question | Answer |
|----------|--------|
| Total batches | 12 |
| Total DRB positive events | 1,608 |
| Total windows | 227,518 |
| Class imbalance | 1:147 |
| Window size / stride / horizon | 250 ms / 50 ms / 500 ms |
| Number of features | 29 |
| Validation AUC-PR | 0.122 |
| Test AUC-PR | 0.070 |
| LOBO mean ± std | 0.102 ± 0.051 |
| Number of C++ patches | 5 |
| RSRP validation accuracy | within 1.2 dBm of ADB |
| Decoder format reverse-engineered | PHY Serving Cell v59 (Snapdragon 8 Gen 2) |

---

## Likely Questions and Pre-Written Answers

**Q: Why didn't you use deep learning?**
A: With 1,608 positive examples, gradient boosting outperforms neural networks. Neural nets need 10,000+ examples to generalize on this kind of imbalanced tabular time-series data. LightGBM also gives me interpretability and sub-millisecond ARM inference for on-device deployment.

**Q: Why AUC-PR not AUC-ROC?**
A: With 1:147 class imbalance, ROC is misleading. A model that always predicts "no reconfig" gets a high AUC-ROC because most windows are negative. AUC-PR measures the precision-recall trade-off, which directly matters for whether a positive prediction is trustworthy.

**Q: How do you know the decoder fix is correct?**
A: Cross-validation against Android ground truth via `dumpsys telephony.registry`. ADB reports RSRP = −96 dBm, my decoder produces −97.12 dBm — within Android's 1 dBm rounding. RSRQ matches exactly. SNR is in the plausible indoor range. Validated across 3 independent packets.

**Q: Why not use the existing Android telephony API?**
A: It only gives 1-second-granularity RSRP/RSRQ. It doesn't expose MAC grants, BSR, HARQ — which are the strongest predictors. The whole point of the diag port is to access data the standard API doesn't expose.

**Q: What's the practical use case?**
A: Three: proactive handover prep (pre-load neighbor params), battery optimization (deeper DRX sleep), and QoS-aware scheduling (pre-allocate buffers before throughput spikes). All require sub-second prediction lead time, which is what 500 ms gives us.

**Q: Will this generalize to other phones?**
A: Not without retraining. The model learns this eNB's vendor algorithm and this chipset's PHY behavior. Cross-device transfer is future work via fine-tuning. The pipeline architecture itself is general.

**Q: Why is test AUC-PR lower than validation?**
A: The test set is small (120 positives in 12,607 windows) and was captured early before some pipeline improvements. Validation reflects current capability; test reflects the lower bound on the older data. Both will improve as more batches are added.

**Q: What's the biggest unknown?**
A: Whether the model can actually generalize across physical locations and times of day. LOBO with std 0.051 suggests reasonable stability across sessions, but I haven't tested across operators or chipsets. That's the main risk for real-world deployment.

---

## Closing Statement (30 seconds)

> *End on the trajectory and what's next.*

To summarize: the pipeline is end-to-end working. The decoder fixes recovered three layers of cellular data that were broken on this chipset. The ML model is generating honest, reproducible metrics that are starting to improve as data accumulates. The work that remains is almost entirely volume-driven, not methodology-driven.

I'm targeting 30 batches over the next 2-3 weeks, then a multi-target training pass to predict the major reconfiguration subtypes simultaneously. After that, the thesis writeup.

The decoder work is reusable for any researcher using MobileInsight on a Snapdragon 8 Gen 2 device — that's a small but real contribution to the open-source community independent of the ML results.
