% Predicting Ciphered LTE Bearer Configuration from Unciphered L2 Headers
% Cross-device and cross-representation results, corrected for collection-position confounding

# 1. Executive summary

## What changed

The initial analysis showed relatively large gains from MAC/RLC features. A later audit found deterministic timing in the collection procedure: 23 fixed-duration `sleep` calls, a fixed 7 s guard interval, predictable activity durations, and capture/scenario alignment. This made event timing correlated with bearer configuration; 79% of UM (voice-bearer) events occurred between 4–13 s into the capture. Raw accuracy therefore cannot be attributed directly to L2 leakage.

## Updated evaluation

To isolate the L2 contribution, the analysis now compares a **content-free (CF) comparator** against **CF + the 19 MAC/RLC features**. The reported side-channel quantity is the incremental gain:

> **L2 increment = Accuracy(CF + L2) − Accuracy(CF)**

This asks whether unciphered L2 metadata provides information beyond timing/event information already available without decoding L2 bytes.

## Main result — strictest valid protocol

Under **leave-one-scenario-out**, with session-clustered confidence intervals and Holm correction over 15 prespecified tests:

| Target | OnePlus | Moto G82 |
|---|---:|---:|
| **RLC mode** | **+2.7 pp** [+1.4, +4.1] | **+2.9 pp** [+1.9, +3.9] |
| LC priority | +3.7 pp [+1.4, +5.9] | +2.5 pp [+1.1, +4.0] |
| LC group | +2.4 pp [+0.4, +4.5] | +2.4 pp [+0.7, +4.2] |
| PDCP discard timer | +0.7 pp [-1.3, +2.8] | +3.4 pp [+1.8, +5.1] |
| **PDCP RoHC** | **+2.2 pp** [+0.9, +3.6] | **+2.5 pp** [+1.5, +3.6] |

**Lead with RLC mode and PDCP RoHC.** RLC mode is the security-relevant target because UM corresponds to the VoLTE voice bearer. The RLC-mode effect is positive and significant on both devices. PDCP RoHC also replicates on both. Do not quote the old leave-one-family-out headline: that protocol is structurally undefined on this dataset.

## Cross-device validation

The result was reproduced on a different Snapdragon device with a different RLC decoder path. The G82 contains 2,394 events versus 1,073 on OnePlus; matched-size subsampling was also performed as corroboration. RLC mode remained positive in 4/5 matched-size runs, so the main result is not simply a consequence of the larger G82 sample.

## Window vs per-packet

On the matched 958-event set, the 500-ms window representation produces substantially larger incremental L2 effects than the per-packet representation. Under scenario-out, RLC mode is **+3.2 pp vs +1.6 pp**, and PDCP RoHC is **+3.9 pp vs +0.3 pp**. The per-packet arm clears the significance bar in 0/15 cells. This supports the interpretation that the useful leakage is primarily an aggregated rate/volume signal, such as grant volume and RLC polling density; windows can also represent periods of silence.

## Bottom line

> **After controlling for deterministic collection timing, unciphered MAC/RLC behavior still provides measurable additional information about ciphered LTE bearer configuration. The strongest replicated result is RLC mode (+2.7/+2.9 pp across the two devices), with a similar replicated effect for PDCP RoHC (+2.2/+2.5 pp).**

The claim is **not** that RRC encryption is broken. The claim is that observable L2 behavior provides a measurable side channel about otherwise ciphered bearer configuration.

## What this document contains

The sections below provide the detailed confound diagnosis, CF construction, statistical protocol, full result tables, cross-device checks, packet-vs-window comparison, and limitations supporting the summary above.

# 2. Why raw accuracy is not the result

Captures were produced by an automated script with **deterministic timing throughout**: the
scenario library contains **23 fixed-duration `sleep` calls and no randomised ones**, a constant
7 s guard band separates activities, and each capture is aligned to the start of its scenario
so that elapsed time is measured from a semantically loaded anchor.

The consequence is direct and measurable. `lib/scenarios.sh` waits exactly 3 s to let the radio
settle and then dials, so the VoLTE bearer appears at a near-fixed offset — **79% of all UM
(voice) events fall between 4 s and 13 s into their capture**:

| seconds into capture | events | UM | UM rate |
|---|---|---|---|
| 0 – 4 s | 424 | 2 | 0.5% |
| **4 – 8 s** | 99 | 55 | **55.6%** |
| **8 – 13 s** | 121 | 56 | **46.3%** |
| 13 – 40 s | 110 | 0 | 0.0% |
| 40 s + | 319 | 27 | 8.5% |

**111 of 140 voice-bearer events (79%) fall in the 4–13 s band**, against a
13.0% overall UM rate — a 3.9× enrichment.

This is not confined to voice: every target is time-locked, with peak-decile class enrichment
of 3.8× to 4.9× (RLC mode UM 4.9×, `none` 4.7×, PDCP RoHC 3.8×, PDCP discard 4.2×).

**Which variable carries it matters.** Elapsed time is the confound; the reconfiguration
ordinal is nearly irrelevant. Predicting RLC mode from each alone (majority class per stratum,
OnePlus, 80.1% baseline): `t_rel` reaches **88.6% (+8.5 pts)**, while `ev_ord` reaches only
**80.4% (+0.3 pts)**. A protocol-driven story — LTE attach installing default bearers before
application traffic — would show up in the ordinal, and it barely does. The dominant cause is
scripted timing, i.e. our collection design, and we state it as such.

A raw accuracy therefore cannot be attributed to the side channel. What *can* be attributed is
the **increment** L2 adds on top of everything obtainable without decoding an L2 byte.

## 2.1 The content-free comparator (CF)

| variable | meaning | available to a real attacker? |
|---|---|---|
| `t_rel` | seconds since the first bearer-adding reconfiguration in this capture | yes, given event detection |
| `ev_ord` | ordinal index of this reconfiguration in its capture | yes, given event detection |
| `tslr` | gap since the previous reconfiguration | yes (past-only) |
| `nwin` | how many windows the pipeline emitted for this event | **no — pipeline artefact** |
| `ev_dur` | time span of those windows | **no — pipeline artefact** |

**Threat-model premise.** The first three assume the attacker knows *that* a reconfiguration
occurred at time *t* — the standard framing, in which the attack predicts the content of an
event it has already detected (e.g. from SRB1 activity). In this pipeline those instants come
from the decrypted RRC stream for ground-truth purposes; an attacker would detect them from
signalling-bearer activity instead.

**CF is computed once per device from the window-level table** and merged onto every arm on
`(batch_id, window_ts)`. This matters: recomputing CF from the per-packet table makes `nwin`
count only the windows that *carried traffic*, disagreeing with the window-level value on
**117 of 958 events (12.2%)**. CF would then absorb traffic presence — the very signal L2 is
meant to supply. After the fix the CF *values* are identical across arms: `t_rel` agrees to
0.000000000 on all 7,004 shared windows.

## 2.2 Is the increment a lower bound? No — and this was measured

`nwin` and `ev_dur` are not attacker-observable, so it is tempting to argue that including
them makes CF stronger than reality and therefore makes every increment a *lower bound*.
**That inference is invalid**, and it is false on this data.

Adding variables to CF raises `CF` and `CF+L2` together, so the increment
`A(CF+L2) − A(CF)` can move either way. Measured on the OnePlus under leave-one-scenario-out,
comparing the reported CF₅ against an attacker-realistic CF₃ = {`t_rel`, `ev_ord`, `tslr`}:

| target | increment with CF₅ (reported) | increment with CF₃ (realistic) | difference |
|---|---|---|---|
| RLC mode (AM/UM/none) | +2.7 | +2.6 | +0.1 |
| LC priority | +3.7 | +2.1 | +1.6 |
| LC group | +2.4 | +4.8 | -2.4 |
| PDCP discard timer | +0.7 | +1.4 | -0.7 |
| PDCP RoHC | +2.2 | +2.1 | +0.1 |

The harsher comparator makes the increment **larger**, not smaller, in 3 of 5 targets. 
Most consequentially, **LC priority** falls from +3.7 to +2.1 once the two pipeline artefacts are removed.

**Consequence for reading this report:** the increments in §4 are measured against CF₅ and
should be quoted as such. Where an attacker-realistic figure is wanted, use CF₃. RLC mode is
essentially unchanged between the two, so the headline claim does not depend on this choice.

# 3. Evaluation protocols

| protocol | what is held out | what it controls | status |
|---|---|---|---|
| leave-one-session-out | one capture session | session/device drift | valid |
| leave-one-scenario-out | every capture of one scenario type | scenario-identity leakage | valid — strictest |
| leave-one-family-out | all call traffic, then all data traffic | traffic-type generalisation | **undefined here** |

**An important clarification.** Leave-one-scenario-out does *not*, by itself, remove the
elapsed-time confound: because every session runs the same fixed recipe, dropping one scenario
leaves the per-capture clock fully represented by the remaining ones. The position control in
this report comes from **CF being in the baseline of every comparison**, not from the fold
assignment. Scenario-out additionally removes scenario-identity leakage, which is why it is
the strictest valid protocol here — but the two mechanisms are separate.

## 3.1 Why leave-one-family-out is undefined for this dataset

This protocol appeared in earlier drafts and is **retracted**. Three verified facts:

1. **`random` sessions dispatch VoLTE calls but were filed as traffic family "data".**
   `run_random_session` adds `activity_call`, `activity_call_burst` and `activity_full_stress`
   to the draw pool. All 19 OnePlus and all 28 G82 `random` batches dispatched such an
   activity; 10/19 and 22/28 respectively went on to record an actual UM voice bearer.
   `activity_full_stress` places a call itself, so `FullStress` is call traffic too.
2. **With the split corrected, the data family is 0.0% UM on both devices.** Every *named*
   data scenario is exactly zero; only `random` was non-zero (9.9% / 10.8%).
3. **Holding out call traffic therefore removes classes entirely.** RLC mode `UM` *is* the
   voice bearer, and class `0` (bearer torn down) occurs only in call sessions.

The degeneracy is not uniform across targets, and the distinction matters:

- For **RLC mode** and **PDCP RoHC** the data-family training set is a *single class*
  (`{AM}` and `{0}`), so the model can only emit a constant. In all four arms these two are
  the cells where CF falls *below* the majority-class baseline — 8 of 8 such cells.
- For **LC priority**, **LC group** and **PDCP discard timer** the training set still has
  2–3 classes, so the model does discriminate among the configurations data traffic produces
  — CF sits *above* baseline there — but the call-only class remains unreachable by
  construction.

Either way the fold is degenerate: in every one of the 20 family-out cells the held-out family
contains a class the training half never saw. That structural test — not any accuracy
comparison — is what flags these cells.

> **The family-out rows in §4 print their numbers for completeness. They must not be quoted.**

This is not a defect in the side channel. It is a structural property of LTE: traffic type and
bearer configuration are entangled by design, so a call-versus-data holdout cannot test this
dataset. `random` genuinely spans both families and is excluded from this protocol only.

## 3.2 Statistics

Reconfiguration events within one session are not independent, and 15 tests are run per arm.
Both are corrected:

- **Session-clustered bootstrap**, 2,000 resamples over *sessions* (not events), giving a 95%
  confidence interval on the increment.
- **Holm–Bonferroni** across all 15 tests within an arm.

A cell counts as a real effect only if the Holm-adjusted *p* < 0.05 **and** the clustered CI
excludes zero. This is stricter than the uncorrected McNemar test used in earlier drafts and
it removes some cells that previously looked significant.

**Which tests the correction runs over — disclosed both ways.** All three protocols were run
before family-out was found to be undefined (§3.1), so the pre-specified family is **15 tests**
and that is what every table below uses. But the 5 family-out cells are structurally incapable
of passing, so including them makes the correction *more* conservative than necessary.
Restricting Holm to the **10 interpretable tests** changes four cells, all from ns to
significant, and none of them on the headline arms:

| arm | cell | p (Holm/15) | p (Holm/10) |
|---|---|---|---|
| OnePlus windowed (958) | session-out PDCP discard timer | 0.1120 (ns) | **0.0420 (sig)** |
| OnePlus windowed (958) | scenario-out LC priority | 0.0658 (ns) | **0.0293 (sig)** |
| OnePlus windowed (958) | scenario-out LC group | 0.1120 (ns) | **0.0420 (sig)** |
| OnePlus **per-packet** | session-out LC priority | 0.0549 (ns) | **0.0366 (sig)** |

**The OnePlus and G82 windowed arms — the arms the headline rests on — are unchanged either
way.** We report the conservative 15-test correction throughout rather than adopting the
variant that yields more significant cells.

## 3.3 Model-seed variance, and why the reported points are single draws

The learner is stochastic: `bagging_fraction = 0.9` and `feature_fraction = 0.8` mean each
tree sees a random subset of events and of features. No seed is set, so LightGBM's default
sub-seeds apply — reproducible, but an arbitrary draw. The session-clustered bootstrap in
§3.2 resamples sessions over a **fixed** fit, so it prices sampling variance but not model
variance. Re-running the identical pipeline over explicit seeds (RLC mode, scenario-out):

| arm | of-record | mean over seeds | sd | range | sign positive |
|---|---|---|---|---|---|
| OnePlus (n=1,073) | +2.70 | **+2.49** | 0.40 | [+1.86, +3.26] | **12/12** |
| Moto G82 (n=2,394) | +2.88 | **+3.23** | 0.13 | [+2.97, +3.43] | **10/10** |

**The increment is positive on 22 of 22 draws across both devices** — the sign never
flips. What is not stable is two-significant-figure precision. Two consequences we state
rather than hide:

- **The reported CIs are narrower than the full uncertainty**, because model variance is
  absent. Folding it in widens the OnePlus interval by roughly ±0.4 pts.
- **The apparent closeness of the two devices is not evidence of agreement.** Over seeds the
  OnePlus sits near +2.5 and the G82 near +3.2 — a ~0.7 pt gap, not the ~0.2 the of-record
  points suggest. Note the G82 of-record figure falls *below every seed tested*, so it
  understates that device rather than flattering it.

We keep the of-record draw in all tables for consistency with the released JSONs, and report
the seed statistics here rather than re-picking a more favourable draw.

# 4. Results

## 4.1 OnePlus · windowed (all 1,073 events)

| target | n | baseline | CF only | CF + L2 | L2 increment | 95% CI (clustered) | p (Holm) | verdict |
|---|---|---|---|---|---|---|---|---|
| **leave-one-session-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 1073 | 80.1% | 90.1% | 92.7% | +2.6 | [+1.4, +3.8] | 0.0008 | **L2 adds** |
| LC priority | 1073 | 44.7% | 77.6% | 82.7% | +5.0 | [+3.5, +6.5] | 0.0000 | **L2 adds** |
| LC group | 1073 | 48.5% | 81.4% | 84.5% | +3.2 | [+1.5, +4.9] | 0.0012 | **L2 adds** |
| PDCP discard timer | 1073 | 51.2% | 77.6% | 80.0% | +2.3 | [+0.6, +4.1] | 0.1488 | ns |
| PDCP RoHC | 1073 | 79.5% | 91.9% | 94.9% | +3.0 | [+1.8, +4.2] | 0.0001 | **L2 adds** |
| **leave-one-scenario-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 1073 | 80.1% | 89.3% | 92.0% | +2.7 | [+1.4, +4.1] | 0.0022 | **L2 adds** |
| LC priority | 1073 | 44.7% | 74.2% | 77.9% | +3.7 | [+1.4, +5.9] | 0.0052 | **L2 adds** |
| LC group | 1073 | 48.5% | 77.6% | 80.1% | +2.4 | [+0.4, +4.5] | 0.1317 | ns |
| PDCP discard timer | 1073 | 51.2% | 76.0% | 76.8% | +0.7 | [-1.3, +2.8] | 0.8750 | ns |
| PDCP RoHC | 1073 | 79.5% | 89.7% | 92.0% | +2.2 | [+0.9, +3.6] | 0.0166 | **L2 adds** |
| **leave-one-family-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 850 | 79.9% | 75.5% | 76.1% | +0.6 | [-0.3, +1.7] | 0.8750 | **undefined** |
| LC priority | 850 | 50.5% | 58.5% | 60.5% | +2.0 | [+0.3, +3.7] | 0.2490 | **undefined** |
| LC group | 850 | 50.5% | 60.6% | 62.0% | +1.4 | [-0.8, +3.6] | 0.8750 | **undefined** |
| PDCP discard timer | 850 | 56.8% | 65.1% | 68.4% | +3.3 | [+0.9, +5.7] | 0.0889 | **undefined** |
| PDCP RoHC | 850 | 78.8% | 74.7% | 74.2% | -0.5 | [-1.1, +0.1] | 0.8750 | **undefined** |

## 4.2 OnePlus · windowed (958-event matched subset)

| target | n | baseline | CF only | CF + L2 | L2 increment | 95% CI (clustered) | p (Holm) | verdict |
|---|---|---|---|---|---|---|---|---|
| **leave-one-session-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 958 | 78.9% | 90.8% | 93.9% | +3.1 | [+1.8, +4.4] | 0.0001 | **L2 adds** |
| LC priority | 958 | 46.7% | 79.2% | 83.2% | +4.0 | [+2.1, +5.8] | 0.0000 | **L2 adds** |
| LC group | 958 | 46.7% | 80.8% | 84.9% | +4.1 | [+2.5, +5.8] | 0.0000 | **L2 adds** |
| PDCP discard timer | 958 | 53.0% | 78.0% | 80.4% | +2.4 | [+0.7, +4.2] | 0.1120 | ns |
| PDCP RoHC | 958 | 78.3% | 92.0% | 95.1% | +3.1 | [+1.9, +4.4] | 0.0001 | **L2 adds** |
| **leave-one-scenario-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 958 | 78.9% | 88.9% | 92.2% | +3.2 | [+1.5, +5.1] | 0.0012 | **L2 adds** |
| LC priority | 958 | 46.7% | 77.0% | 79.9% | +2.8 | [+0.7, +4.7] | 0.0658 | ns |
| LC group | 958 | 46.7% | 77.2% | 79.5% | +2.3 | [+0.6, +4.0] | 0.1120 | ns |
| PDCP discard timer | 958 | 53.0% | 76.1% | 76.6% | +0.5 | [-1.4, +2.4] | 1.0000 | ns |
| PDCP RoHC | 958 | 78.3% | 89.4% | 93.2% | +3.9 | [+2.5, +5.4] | 0.0000 | **L2 adds** |
| **leave-one-family-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 768 | 78.8% | 76.0% | 75.7% | -0.4 | [-1.2, +0.4] | 1.0000 | **undefined** |
| LC priority | 768 | 51.8% | 59.8% | 61.8% | +2.1 | [+0.3, +3.8] | 0.1171 | **undefined** |
| LC group | 768 | 51.8% | 62.0% | 62.9% | +0.9 | [-0.8, +2.5] | 1.0000 | **undefined** |
| PDCP discard timer | 768 | 58.2% | 67.7% | 68.6% | +0.9 | [-1.0, +2.9] | 1.0000 | **undefined** |
| PDCP RoHC | 768 | 77.7% | 74.3% | 74.1% | -0.3 | [-1.1, +0.6] | 1.0000 | **undefined** |

## 4.3 Moto G82 · windowed

| target | n | baseline | CF only | CF + L2 | L2 increment | 95% CI (clustered) | p (Holm) | verdict |
|---|---|---|---|---|---|---|---|---|
| **leave-one-session-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 2394 | 79.8% | 88.6% | 91.7% | +3.0 | [+2.1, +4.0] | 0.0000 | **L2 adds** |
| LC priority | 2394 | 42.5% | 76.9% | 80.5% | +3.5 | [+2.2, +5.0] | 0.0000 | **L2 adds** |
| LC group | 2394 | 49.7% | 80.9% | 84.2% | +3.3 | [+2.0, +4.6] | 0.0000 | **L2 adds** |
| PDCP discard timer | 2394 | 51.0% | 74.4% | 78.1% | +3.7 | [+2.2, +5.1] | 0.0000 | **L2 adds** |
| PDCP RoHC | 2394 | 80.1% | 88.3% | 91.6% | +3.3 | [+1.9, +4.7] | 0.0000 | **L2 adds** |
| **leave-one-scenario-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 2394 | 79.8% | 88.1% | 90.9% | +2.9 | [+1.9, +3.9] | 0.0000 | **L2 adds** |
| LC priority | 2394 | 42.5% | 73.8% | 76.4% | +2.5 | [+1.1, +4.0] | 0.0004 | **L2 adds** |
| LC group | 2394 | 49.7% | 77.2% | 79.7% | +2.4 | [+0.7, +4.2] | 0.0031 | **L2 adds** |
| PDCP discard timer | 2394 | 51.0% | 70.6% | 74.0% | +3.4 | [+1.8, +5.1] | 0.0000 | **L2 adds** |
| PDCP RoHC | 2394 | 80.1% | 87.3% | 89.8% | +2.5 | [+1.5, +3.6] | 0.0000 | **L2 adds** |
| **leave-one-family-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 1828 | 80.0% | 76.4% | 77.2% | +0.9 | [+0.4, +1.4] | 0.0031 | **undefined** |
| LC priority | 1828 | 52.1% | 57.0% | 60.9% | +3.9 | [+2.3, +5.5] | 0.0000 | **undefined** |
| LC group | 1828 | 52.1% | 58.6% | 62.6% | +4.0 | [+2.3, +5.6] | 0.0000 | **undefined** |
| PDCP discard timer | 1828 | 60.2% | 64.0% | 70.0% | +6.0 | [+4.5, +7.6] | 0.0000 | **undefined** |
| PDCP RoHC | 1828 | 78.3% | 73.8% | 75.8% | +2.0 | [+1.2, +3.0] | 0.0000 | **undefined** |

## 4.4 OnePlus · per-packet (958 events)

| target | n | baseline | CF only | CF + L2 | L2 increment | 95% CI (clustered) | p (Holm) | verdict |
|---|---|---|---|---|---|---|---|---|
| **leave-one-session-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 958 | 78.9% | 87.8% | 89.4% | +1.6 | [+0.4, +2.7] | 0.1759 | ns |
| LC priority | 958 | 46.7% | 77.5% | 79.5% | +2.1 | [+0.9, +3.3] | 0.0549 | ns |
| LC group | 958 | 46.7% | 81.4% | 82.6% | +1.1 | [+0.2, +2.1] | 0.4163 | ns |
| PDCP discard timer | 958 | 53.0% | 78.2% | 79.0% | +0.8 | [-0.5, +2.1] | 1.0000 | ns |
| PDCP RoHC | 958 | 78.3% | 88.6% | 89.9% | +1.3 | [+0.2, +2.5] | 0.6337 | ns |
| **leave-one-scenario-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 958 | 78.9% | 87.5% | 89.0% | +1.6 | [+0.5, +2.7] | 0.1494 | ns |
| LC priority | 958 | 46.7% | 76.1% | 76.5% | +0.4 | [-0.9, +1.7] | 1.0000 | ns |
| LC group | 958 | 46.7% | 77.0% | 77.1% | +0.1 | [-1.0, +1.3] | 1.0000 | ns |
| PDCP discard timer | 958 | 53.0% | 78.7% | 77.7% | -1.0 | [-2.3, +0.3] | 1.0000 | L2 hurts |
| PDCP RoHC | 958 | 78.3% | 88.5% | 88.8% | +0.3 | [-0.6, +1.3] | 1.0000 | ns |
| **leave-one-family-out** | | | | | | | | |
| RLC mode (AM/UM/none) | 768 | 78.8% | 74.1% | 74.1% | +0.0 | [-0.9, +0.8] | 1.0000 | **undefined** |
| LC priority | 768 | 51.8% | 59.5% | 59.9% | +0.4 | [-0.6, +1.4] | 1.0000 | **undefined** |
| LC group | 768 | 51.8% | 61.2% | 61.3% | +0.1 | [-0.9, +1.2] | 1.0000 | **undefined** |
| PDCP discard timer | 768 | 58.2% | 67.6% | 68.6% | +1.0 | [-0.3, +2.4] | 1.0000 | **undefined** |
| PDCP RoHC | 768 | 77.7% | 73.4% | 73.6% | +0.1 | [-0.3, +0.6] | 1.0000 | **undefined** |

# 5. Cross-device comparison (windowed representation)

Under **leave-one-scenario-out**:

| target | OnePlus (n=1,073) | Moto G82 (n=2,394) | significant on |
|---|---|---|---|
| RLC mode (AM/UM/none) | +2.7 [+1.4, +4.1] | +2.9 [+1.9, +3.9] | **both** |
| LC priority | +3.7 [+1.4, +5.9] | +2.5 [+1.1, +4.0] | **both** |
| LC group | +2.4 [+0.4, +4.5] | +2.4 [+0.7, +4.2] | G82 only |
| PDCP discard timer | +0.7 [-1.3, +2.8] | +3.4 [+1.8, +5.1] | G82 only |
| PDCP RoHC | +2.2 [+0.9, +3.6] | +2.5 [+1.5, +3.6] | **both** |

The column says *significant on*, not *replicates on*: the two are not the same thing.
For **LC group** the two point estimates are close (+2.4 vs
+2.4) with overlapping CIs, and the OnePlus cell fails only the
Holm-adjusted threshold. We do **not** read the two-decimal agreement as meaningful: it is
well below the model-seed noise floor of ±0.4 pts measured in §3.3. LC group also fails to
survive matched-sample subsampling (§5.1, 0 of 5 seeds), so we do not claim it as
cross-device replicated.

The two devices use different chipsets and require different RLC decoder paths (the G82 needs
its own format dispatch), so agreement here is a genuine replication rather than a re-run.

## 5.1 Matched sample size — is the cross-device agreement a size artefact?

The G82 set has 2,394 events against the OnePlus's 1,073. A larger sample does not inflate
an effect *size*, but it does tighten confidence intervals and therefore makes significance
easier to reach. To remove that as an explanation, the G82 arm was re-run on **exactly
1,073 randomly chosen events** (5 independent seeds), through the identical
pipeline: same CF, same protocol, same model, same session-clustered CIs, same Holm
correction (see the note below on how many tests it corrects over). Events (not rows) are
subsampled, so every window of a kept event stays together and the fold structure is
unchanged.

| target | OnePlus (n=1,073) | G82 full (n=2,394) | **G82 @ n=1,073** | holds at matched n? |
|---|---|---|---|---|
| RLC mode (AM/UM/none) | +2.7 [+1.4, +4.1] | +2.9 [+1.9, +3.9] | **+2.0 (4/5 sig)** | partly |
| LC priority | +3.7 [+1.4, +5.9] | +2.5 [+1.1, +4.0] | **+2.7 (2/5 sig)** | partly |
| LC group | +2.4 [+0.4, +4.5] | +2.4 [+0.7, +4.2] | **+0.7 (0/5 sig)** | **no** |
| PDCP discard timer | +0.7 [-1.3, +2.8] | +3.4 [+1.8, +5.1] | **+5.5 (5/5 sig)** | **yes** |
| PDCP RoHC | +2.2 [+0.9, +3.6] | +2.5 [+1.5, +3.6] | **+1.9 (3/5 sig)** | partly |

**A note on the significance bar, stated so it is not mistaken for the main table's.**
"sig" here means Holm-adjusted *p* < 0.05 **and** a clustered CI excluding zero — the same
two conditions used everywhere else. But this sub-study runs one protocol, so Holm corrects
over its **5 tests**, whereas the main tables correct over **15**. That is the correct
correction for the family of tests actually performed here, and it is a slightly easier bar
than the main table's. Read the matched-n column as corroboration of the cross-device
result, not as a drop-in replacement for it.

Subsampling also removes whole captures: 1,073 events come from ~352 of the 379 batches
across ~23 of 24 scenarios, since small batches lose all their events. Each seed therefore
sees a slightly different scenario set, which is why several seeds are run rather than one.

# 6. Cross-representation comparison (windowed vs per-packet)

Both arms use the **same 958 events, the same fold definitions, the same CF variable values**
(max difference 0 across all 7,004 shared windows), **the same hyperparameters and the same
statistics**.

Two things beyond the representation do differ, and both are consequences of it rather than
confounds that could be removed:

- the per-packet arm has ~80× more rows, because each window's CF row is repeated once per
  packet in that window — so training is implicitly weighted by packet count;
- the per-packet arm has 80 L2 features against the windowed arm's 19.

**leave-one-session-out**

| target | windowed (19 feat) | per-packet (80 feat) | difference |
|---|---|---|---|
| RLC mode (AM/UM/none) | +3.1 [+1.8, +4.4] | +1.6 [+0.4, +2.7] | -1.6 |
| LC priority | +4.0 [+2.1, +5.8] | +2.1 [+0.9, +3.3] | -1.9 |
| LC group | +4.1 [+2.5, +5.8] | +1.1 [+0.2, +2.1] | -2.9 |
| PDCP discard timer | +2.4 [+0.7, +4.2] | +0.8 [-0.5, +2.1] | -1.6 |
| PDCP RoHC | +3.1 [+1.9, +4.4] | +1.3 [+0.2, +2.5] | -1.9 |

**leave-one-scenario-out**

| target | windowed (19 feat) | per-packet (80 feat) | difference |
|---|---|---|---|
| RLC mode (AM/UM/none) | +3.2 [+1.5, +5.1] | +1.6 [+0.5, +2.7] | -1.7 |
| LC priority | +2.8 [+0.7, +4.7] | +0.4 [-0.9, +1.7] | -2.4 |
| LC group | +2.3 [+0.6, +4.0] | +0.1 [-1.0, +1.3] | -2.2 |
| PDCP discard timer | +0.5 [-1.4, +2.4] | -1.0 [-2.3, +0.3] | -1.6 |
| PDCP RoHC | +3.9 [+2.5, +5.4] | +0.3 [-0.6, +1.3] | -3.5 |

*The difference column is computed from unrounded increments, so it may differ by 0.1 from
subtracting the two displayed values.*

**The per-packet arm clears the significance bar in 0 of 15 cells** under the
pre-specified correction, and **1 of 10** if Holm is restricted to the
interpretable tests (§3.2). Either way it is far below the windowed arm, which clears 2 of
10 on the same events with the same comparator.

## 6.1 Is the per-packet arm simply facing a stronger baseline?

For RLC mode under leave-one-scenario-out, no:

| arm | CF only | CF + L2 | increment |
|---|---|---|---|
| windowed | 88.9% | 92.2% | **+3.2** |
| per-packet | 87.5% | 89.0% | **+1.6** |

The two CF baselines are within 1.5 points and the per-packet
one is the *lower* of the two, so its smaller increment is not a baseline artefact here.

**This is not uniform.** Across the 10 cells in §6, the per-packet CF baseline is *higher* in
3. For PDCP discard timer under scenario-out in particular the per-packet CF is
the higher of the two, so that cell's negative increment should not be read as the
representation losing information.

## 6.2 Why the per-packet representation recovers less

**A caveat on capacity first.** The §6 numbers use the of-record configuration — 15 leaves,
120 trees — on *both* arms. A separate 3-fold sweep on L2 features alone
(`outputs/packet_capacity_audit.json`) shows the per-packet arm is capacity-sensitive while
the windowed arm is saturated. The per-packet increments here may therefore be understated,
and this comparison should be read as "at equal, of-record capacity" rather than "at each
arm's best".

With that stated, four measured structural reasons:

1. **The leak is a rate, not a per-packet property.** The dominant channel is uplink grant
   volume and RLC polling density — quantities that exist only once packets are counted over
   an interval. A single packet does not carry them.
2. **A window can represent silence; a packet cannot.** Inside the matched 958 events, 428 of
   the windowed arm's 7,432 rows (5.8%, touching 117 events) are all-zero windows that the
   per-packet table cannot represent at all — there is no packet to make a row from. Those
   windows are UM-enriched, which is exactly where the polling-absence signal lives.
3. **The presence flags carry no information.** Each parser returns a fixed key set per
   record, so all **31** `has_*` flags collapse to a 4-way layer one-hot.
4. **More rows are not more samples.** Statistical power is set by the 958 events, not the
   595,903 packet rows. Those rows carry only 118,267 distinct PDUs — about **5** copies per
   PDU (median 5, max 10) from the 50 ms stride — which in turn describe just 958 events.

# 7. Limitations, stated plainly

1. **Collection order is confounded with configuration.** Every regime runs a fixed scenario
   order, so capture position predicts the label. This is why every number here is an
   increment over CF rather than an accuracy. The fix for future collection is one line:
   randomise scenario order.
2. **Absolute accuracies are mostly not the side channel.** Under leave-one-scenario-out on
   the OnePlus, CF alone reaches 89.3% on RLC mode against a 92.0% combined figure. Quoting
   the 92% as a side-channel result would be wrong; the defensible quantity is the gap.
3. **The increments are not lower bounds** (§2.2). They are measured against CF₅, which
   includes two variables an attacker does not have. For LC priority the attacker-realistic
   figure is materially smaller.
4. **Five targets are not five independent findings.** RLC mode and LC group are exact
   deterministic functions of LC priority on both devices (`0→none, 3→UM, 4→AM, 8→AM`).
   PDCP discard timer and PDCP RoHC are not exact functions of it, but are close — LC priority
   alone predicts them at roughly 91–96%. Treat the study as **one finely-resolved quantity
   (LC priority) plus two near-redundant coarsenings**, not as five independent leaks.
5. **Leave-one-family-out cannot be run on this dataset** (§3.1).
6. **Equal-capacity comparison** in §6, not each arm's best (§6.2).
7. **Stationary, single-cell, 4G captures.** Handover and EN-DC configurations are absent, and
   there is no per-packet arm for the G82.

