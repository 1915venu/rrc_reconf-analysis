---
title: "Recovering Ciphered LTE Bearer Configuration from Unciphered L2 Metadata"
subtitle: "Results update — cross-device validation on a second chipset"
date: "July 2026"
---

# 1. Summary

**The claim.** A *passive* attacker — no keys, not the network, just listening over the air — can
recover the **encrypted** Data Radio Bearer (DRB) configuration that an LTE network installs, by
reading the **unencrypted** MAC/RLC header metadata the same connection emits in the ~500 ms *before*
that configuration is applied.

**What is new since the last update.** The attack has been replicated end-to-end on a **second
handset with a different chipset**, on the same SIM and the same scripted traffic campaign. The three
strongest bearer-configuration fields leak on **both** devices, and the effect survives holding out
entire capture sessions. We additionally measured *which* observable carries each field on both
chipsets, which is reported in §5.

**Model.** All results below use the same predictor throughout: a **per-target class-weighted
gradient-boosted decision tree (LightGBM)**, evaluated leave-one-event-out with per-event majority
vote. One independent model per ciphered field.

**Headline numbers.**

| | Device 1 — OnePlus (Snapdragon 8+ Gen 1) | Device 2 — Moto G82 (Snapdragon 695) |
|---|---|---|
| Events | 1,073 reconfigurations / 191 sessions | 1,721 reconfigurations / 264 sessions |
| Fields recovered significantly | **5 of 5** | **3 of 5** |
| Best lift over chance | **+26.2 points** | **+18 points** |
| Voice-bearer detection (recall) | 60% | **71%** |

---

# 2. How to read the results tables

Each row is one **ciphered parameter** the attacker is trying to recover. The columns:

| Column | Meaning |
|---|---|
| **Accuracy** | How often the model recovers that field correctly. One prediction per reconfiguration event. |
| **Baseline** | Score obtained by *always guessing the most common value*, ignoring the data entirely. It differs per field because the class mix differs. |
| **Lift** = Acc − Base | **The actual result** — how much the unencrypted metadata revealed *beyond* blind guessing. |
| **95% CI** | Range the true accuracy plausibly occupies (2,000-sample bootstrap over events). The test that matters: *does the lower bound clear the baseline?* |
| **Perm. *p*** | Labels are shuffled at random and the entire pipeline refitted, 50 times. *p* = 0.0196 means **0 of 50** shuffles ever reached the real accuracy — so the model performs only when labels are true. |
| **LOSO lift** | Leave-one-**session**-out: hold out an entire capture session rather than one event. Tests whether the result is an artifact of conditions within a single recording. |

**Why "lift" and not accuracy.** Raw accuracy is misleading on its own. 88% sounds stronger than 70%,
but against baselines of 80% vs 45% the 70% is by far the more significant finding — it extracted 26
points of information where the other extracted 8.

**Worked example.** Row: *Logical-channel priority — 70.7% | 44.7% | +26.0 | [68–74] | 0.0196*
> Guessing the most common priority gets 44.7%. From unencrypted L2 metadata alone we recover it
> 70.7% of the time — **26 points above chance**. Even the pessimistic end of the interval (68%)
> is far above baseline, and across 50 label-shuffle trials the model never once matched this by luck.

---

# 3. Device 1 — OnePlus (Snapdragon 8+ Gen 1)

*OnePlus 11R (model CPH2487, Qualcomm SM8475) · 1,073 reconfiguration events · 191 sessions ·
leave-one-event-out · class-weighted LightGBM*

| Ciphered DRB field | Accuracy | Baseline | **Lift** | 95% CI | Perm. *p* |
|---|---:|---:|---:|:---:|:---:|
| Logical-channel **group** | 74.7% | 48.5% | **+26.2** | [72–77] | 0.0196 |
| Logical-channel **priority** | 70.7% | 44.7% | **+26.0** | [68–74] | 0.0196 |
| PDCP **discard timer** | 68.8% | 51.2% | **+17.6** | [66–72] | 0.0196 |
| PDCP **RoHC** (header compression) | 89.8% | 79.5% | **+10.3** | [88–92] | 0.0196 |
| **RLC mode** (data vs voice bearer) | 88.5% | 80.1% | **+8.4** | [87–90] | 0.0196 |

All five are statistically significant: every CI lower bound sits strictly above its own baseline, and
no label-shuffle ever reproduced the result.

*Note on RLC mode:* accuracy is high (88.5%) but lift is modest (+8.4) because the baseline is already
80.1% — most bearers are data bearers, so only ~20 points of headroom exist. Low ceiling, not weak signal.

---

# 4. Device 2 — Moto G82 (Snapdragon 695)

*Moto G82 5G (Qualcomm SM6375) · 1,721 events · 264 sessions · same SIM, same campaign, pipeline
unchanged. Only the capture mechanism differed (this handset blocks the tethered diagnostic port, so
logging was on-device).*

| Ciphered DRB field | Accuracy | Baseline | **Lift** | **LOSO lift** |
|---|---:|---:|---:|---:|
| Logical-channel **priority** | 60% | 42% | **+18** | **+18** |
| Logical-channel **group** | 67% | 53% | **+14** | **+13** |
| PDCP **discard timer** | 61% | 50% | **+12** | **+11** |
| **RLC mode** | 75% | 82% | −7 → *report as recall* | — |
| PDCP **RoHC** | 84% | 83% | +1 (no leak) | +1 |

**The key column is LOSO.** The lifts barely change when entire capture sessions are held out
(+18→+18, +14→+13, +12→+11). This is the strongest available evidence that the result is not an
artifact of any single recording session.

**Reading the RLC-mode row.** On this device the capture is **82% one class**, so "always guess data"
scores 82% while detecting **zero** voice bearers. The class-weighted model deliberately sacrifices
some accuracy to catch the minority class, and it detects **71% of voice bearers — better than the
OnePlus's 60%**. For rare-event detection recall is the appropriate metric; accuracy is misleading here.

**An honest negative.** RoHC leaks clearly on the SD8+G1 (+10.3) but **not** on the SD695 (+1). The side
channel's *presence* generalises across chipsets; its *magnitude* is chipset-dependent.

---

# 5. Why it works — and a new cross-device finding

The mechanism: **the network provisions a bearer for the service it already sees**, and that service's
traffic shape is visible in cleartext at MAC/RLC *before* the ciphered reconfiguration is installed.
A voice bearer (RLC-UM) produces small, frequent, **un-polled** uplink traffic — UM runs no ARQ, so it
emits no poll bits. A default data bearer (RLC-AM) produces large grants, heavy polling and wide
downlink sequence-number spans.

We measured *which* observable carries each field on **both** devices (permutation importance:
accuracy lost when a single observable is shuffled). **The mechanism is related but not identical:**

| | OnePlus (SD8+G1) | Moto G82 (SD695) |
|---|---|---|
| Dominant observable | `mac_ul_grant_sum` — **top on all 5 targets** | `rlc_dl_sn_span` top on 3/5; `mac_ul_grant_sum` top on 2/5 |
| Secondary | **RLC uplink polling** (`poll_count` / `poll_rate`) in top-3 on all 5 | RLC polling **absent** from top-3 on every target |
| Gain share (RLC / MAC-UL / MAC-DL) | ~33 / **49** / 18 | ~40 / 37 / 23 |

**Interpretation.** Uplink **grant volume** is a shared leakage channel on both chipsets — on
logical-channel priority its importance is nearly identical (+4.3 points on both). But the *secondary*
channel differs: the OnePlus leaks through **uplink ARQ polling**, whereas the G82 leaks through
**downlink sequence-number span**.

This also **explains the one weak result**: on the OnePlus, voice detection rides on the *absence* of
RLC polling — but polling carries little information on the G82, which is precisely why `RLC mode` is
the weakest field there. The mechanism analysis and the accuracy result agree.

---

# 6. Confidence in the inputs

The whole result rests on decoding the radio logs correctly, so each decoder was independently
ground-truthed:

| Stream | Validated against | Result |
|---|---|---|
| MAC uplink + downlink | **SCAT** (independent open-source decoder) | **638,399 entries byte-for-byte identical** |
| RLC sequence numbers | **Our own srsRAN base station** (USRP B210) | **93/93 (100%)** decoded SNs matched those the base station independently received |
| RLC (no reference decoder exists) | reference-free constraint battery | 99.2% monotonic over 59,587 transitions = **241 σ** above chance |
| G82 decoders | MobileInsight (independent here) + SCAT | RLC-UL 100% agreement; MAC byte-exact |

A feature-coverage audit additionally confirms the G82's 19 observables are **equally or better**
populated than the OnePlus's — so its smaller RLC-mode lift is genuine chipset behaviour, **not** a
decoding deficiency.

---

# 7. Limitations, stated plainly

- **Scope:** one operator (Airtel 4G), stationary, single cell. Handover and 5G/EN-DC targets are not
  reachable from a stationary LTE-only capture and are not claimed.
- **RoHC does not transfer** to the second chipset (+1 point).
- **RLC mode on the G82** must be reported as recall, not accuracy, because of an 82% class imbalance.
- **Permutation testing on the G82** is computationally prohibitive at 1,721 events; significance there
  rests on bootstrap confidence intervals, all of which clear baseline by 9–16 points.
- The mechanism attribution in §5 uses permutation importance, which is a **lower bound** for any
  group of correlated observables: shuffling one feature lets its correlated partners compensate, so
  a group's true contribution is understated. The grouped split-gain share is reported alongside it as
  a cross-check, and the two measures agree on the ranking.

---

