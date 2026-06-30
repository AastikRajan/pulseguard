# PulseGuard — THE Definitive Benchmark (project conclusion)

**Date:** 2026-06-30 · **Status:** research prototype, NOT clinical software · **Honesty bar:** brutal, under-claimed.
**Fresh evaluation:** `scripts/build_final_benchmark.py` → `outputs/world_model/benchmark_final.json` (full locked
test set: 303,909 windows / 1,030 cases, split `rng(2026)` 20%-by-case). Cross-site/causal/lead-time numbers
cited from their validated source evals. This file supersedes the 2026-06-28 `PROJECT_BENCHMARK.md` as the
consolidated scoreboard; entry point is `docs/PULSEGUARD_README.md`.

---

## 0. One paragraph
PulseGuard is a multi-layer intraoperative intelligence **system** on VitalDB (5,151 cases, Seoul National
University Hospital, 35 channels @1Hz + 500Hz waveforms). It turns streaming vitals into **clinical
intelligence** — *which* of 9 deteriorations is coming, *why* (physiology + model), *how worried to be for this
patient*, and *what to do* — with calibrated alarm precision, ~4.5-min lead on the events that have precursors,
and zero-shot cross-hospital generalization. It is held to a strict honesty bar (locked by-case split, leakage
quarantine, no gray-zone exclusion, nulls reported), and it ships with a **rigorously-mapped predictability
boundary**: it states precisely what it can and cannot predict, and why. A separate causal arm shows the
intraoperative-hypotension→AKI association is ~90% confounding by indication, replicated on a second hospital.

---

## 1. Data & honesty bar
- **Dataset:** VitalDB (SNUH), 5,151 cases / 1,574,393 windows / 105,596 labeled events. 35 channels @1Hz
  (circulation, full ventilator, oxygenation, BIS/depth, temperature, 4 drug-infusion rates) + 500Hz waveforms.
- **Locked test split:** `rng(2026)`, 20% by case — locked before model selection, identical across all arms.
- **Leakage quarantine:** the pre-2026-06-27 `run_asym_strict_v4_event_rescue.py` lineage is banned (label leak).
- **No gray-zone exclusion** (the dominant source of published AUROC inflation; Enevoldsen & Vistisen 2022).
- **In-sample honesty:** the *deployed* `production_ews_v2` (trained on all cases) scores AUPRC 0.48 on the test
  20% = **in-sample inflated**. The **honest held-out number (clean dev-only train) is 0.454** — reported below.

---

## 2. Early-Warning System — the deployable core

### 2a. Unified "any critical deterioration in 5 min" (honest held-out, fresh)
| Metric | Value |
|---|---|
| AUPRC (honest held-out) | **0.454** (validated stack: 0.452 multi · 0.455 event-history · 0.466 SSL) |
| AUROC | **0.849** |
| base rate | 0.123 |
| @0.5 FA/hr | sens 0.072 · **PPV 0.71** |
| @1.0 FA/hr | sens 0.12 · **PPV 0.67** |
| @2.0 FA/hr | sens 0.19 · **PPV 0.62** |
| Lead time @1 FA/hr | ~4.5 min |

### 2b. Per-event EWS (all 9 events, clean dev-only, full test)
| Event | base | AUPRC | AUROC | lift | note |
|---|---|---|---|---|---|
| hypoxemia | 0.031 | 0.416 | **0.957** | 13.3× | rich precursors (vent-mechanics lead) |
| light_anesth (awareness) | 0.048 | 0.449 | **0.932** | 9.4× | ★ new capability, drug-anticipated |
| apnea | 0.029 | 0.342 | 0.949 | 11.9× | + deterministic circuit-integrity rule |
| tachycardia | 0.028 | 0.299 | 0.950 | 10.9× | |
| ischemia (ST-deviation) | 0.249 | 0.436 | 0.743 | 1.8× | non-specific cardiac-stress flag |
| bronchospasm | 0.006 | 0.119 | 0.936 | 18.5× | |
| bradycardia | 0.006 | 0.184 | 0.945 | 30.5× | |
| hypercapnia | 0.002 | 0.031 | 0.942 | 18.0× | rare |
| **hypotension** | 0.076 | **0.243** | **0.822** | 3.2× | **the ceiling — see §7** |

Gains track physiology: highest where reserves create multi-signal precursors (hypoxemia/tachycardia), lowest
for hypotension (no usable precursor — §7).

### 2c. Alarm quality (the field's #1 problem = alarm fatigue)
- **Conformal tiered alarms** — calibrated false-alarm control (target 0.5/1.0/2.0 → achieved 0.54/1.10/2.12 per
  case-hour). **PPV 0.61–0.71** at deployable operating points vs ~0.26 naive threshold = **~7× the field's
  deployed PPV (~0.12)**.
- **Episode-grouping** (hysteresis + merge): **0.85 episode-precision** (HIGH-tier 0.89), **308× alarm reduction**.
- **Circuit-integrity / apnea L0 rule** (ETCO₂-flatline + no-TV): **3.2-min lead before SpO₂ falls, 0.89% FA** —
  a deterministic Tier-1 "check circuit/airway" alert distinct from desaturation.

---

## 3. The system (what makes it more than a model)
8 layers, 4 deployable surfaces (demo HTML · co-pilot timeline UI · callable API · — HTTP server removed per scope):
L0 rules (incl. circuit-integrity) · L1 EWS risk · L2 **differential-dx** (6-way class-balanced, **macro-recall
0.61**) · L3 context-severity + biology + **surgical-plan context** · L4 conformal PPV tiers · L5 narrative + SHAP ·
L6 stateful episode-grouping · context-conditioned operating point. **9-event monitor:** the 6-way differential +
independent detectors for **awareness (AUROC 0.93)** + **ischemia (0.74)** + the apnea rule (a naive 9-way
differential degrades the 6 from 0.61→0.454 because events co-occur → independent detectors are the right design).

---

## 4. Cross-hospital transportability (zero-shot, MOVER / UC-Irvine)
- **Hypoxemia EWS** (HR+SpO₂, 1-min): internal AUROC **0.843 → external 0.782** — modest drop across
  country/monitors/charting; AUPRC drop is base-rate (6× rarer); lift ~7.4×.
- **EPIC hypotension EWS** (continuous arterial MAP): internal 0.839 → external **0.802**.
- **Transportability principle:** universally-DENSE channels transport (HR/SpO₂/continuous-MAP); rich
  source-specific channels (sparse NIBP, ETCO₂ cascade) HELP in-domain but HURT zero-shot transfer. Mechanistic
  root (§6 robustness): HR/SpO₂ are the irreducible, no-proxy channels. **Per-site calibration:** the conformal
  FA-rate transports as-is (calibrated on negatives, base-rate-invariant); only PPV-targeting needs a local
  threshold trade. The machinery is NOT Seoul-specific.

## 5. Causal IOH → AKI deconfounding (the second deliverable)
- VitalDB n=2,730 (AKI 10.3%). Naive RR (AUC-MAP<65, p90 vs p10) **1.77 → g-formula 1.07** (**90% shrinkage**,
  CI crosses 1.0); negative-control + TIVA-restricted sensitivity all clean/null. E-values reported.
- **Replicated on a 2nd hospital** (EPIC/UC-Irvine, n=6,831): naive 1.30 → NCO-calibrated **0.98 (null)** — the
  surviving association is proven confounding by an equally-large negative-control bias. **Two countries, same
  conclusion: IOH→AKI is confounding by indication** — consistent with all three null 2025 prevention RCTs.

## 6. Robustness, outcomes & context
- **Graceful degradation; 4 irreducible must-measure channels** = HR, SpO₂, pressure line, ventilator. Outside a
  total pressure dropout (hits hypotension −0.053) or ventilator dropout (hits hypoxemia −0.095), no missing
  channel group costs >~0.02 AUPRC. **HR is the most unique** (R² 0.30 recoverable, no proxy) — the mechanistic
  root of the transportability principle.
- **Outcome prediction:** intraop course → ICU admission AUROC **0.928** (vs 0.746 preop-only) — the prognostic
  signal is genuinely physiological.
- **Surgical-plan context** (chi² p<0.001, anticipatory): major-resection → hypotension 88% vs 70%; spinal → 21%
  (protective); lateral decubitus → 3× hypercapnia; Trendelenburg → tachycardia. Belongs in the severity layer;
  multivariate preop fusion adds small-real lift for hemodynamic events only (+0.007), hurts awareness.

---

## 7. The mapped boundary (the rigor that is the contribution)
- **Hypotension is ceiling-bound, confirmed 7 INDEPENDENT WAYS** — operating-point · PI/PPV/EaDyn lead-signals ·
  univariate CSD · multivariate DNB · short-horizon (reachability-collapse) · phenotyping (homogeneous continuum) ·
  autonomic 500Hz HRV/BPV. Mechanism: a slow, homogeneous, blunted-baroreflex drift whose autonomic precursor is
  self-erased by anesthesia. System recall 0.12 @high-precision (tunable to ~0.36). **Not a bug — the physiology.**
- **The differential is non-specific early:** MAP/RR/SEF/compliance move first before *every* event → you can tell
  *something* is coming, not *what*; the differential resolves only as the event nears.
- **Deterioration syndromes:** hypotension↔ST-stress 47% (demand ischemia); ST-deviation is a non-specific stress
  hub, not confirmed coronary ischemia.
- **Re-confirmed honest nulls:** Chronos-2 / MOMENT / PaPaGei foundation-model forecasters · waveform morphology
  (all 5 approaches) · unsupervised world-model anomaly · treatment-aware drug-residual (both drugs, untimed-bolus
  wall) · drug-rate features add ~0 to the EWS (feedback-confounded) · preop labs ≈ chance (intraop-driven).

---

## 8. SOTA positioning & the honest contribution
- **vs deployed monitors:** ~7× the PPV at deployable false-alarm rates; 308× alarm reduction; multi-signal
  precursor lead vs single-threshold reaction.
- **vs published IOH models:** comparable-or-better *without* the gray-zone exclusion that inflates them.
- **What's genuinely novel:** the cross-continental transportability *science* (what transports & why); the
  exhaustively-mapped hypotension ceiling (a rigorous negative); the leak-free event-history features (+0.011, zero
  inference cost); awareness monitoring as a drug-anticipated capability; the integration into a calibrated,
  interpretable *system*.
- **Honest scope:** an *integration + interpretability + rigor + boundary-mapping* contribution — NOT a
  new-signal breakthrough, and NOT clinical software. The discipline that refuted its own marketing (caught leaks,
  in-sample inflation, autocorrelation, subagent bugs) is the moat.

**Bottom line:** a 9-event, context-aware, cross-hospital-generalizable, calibrated early-warning **system** with
honest held-out AUPRC 0.454 / AUROC 0.849, ~7× deployed PPV, ~4.5-min lead, a second causal deliverable, and a
predictability boundary mapped to the point of exhaustion. Everything here is reproducible from the locked split.
