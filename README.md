# PulseGuard — early warning for the operating room

### ▶ Live demo: **https://aastikrajan.github.io/pulseguard/**

Software that reads the live vital signs during surgery and warns about **4 minutes before** the patient
gets into trouble — and says *which* problem (out of 9) and *why*. A research prototype, **not** a medical device.

Built solo by **Aastik Rajan** (Johns Hopkins University · aastikc15@gmail.com) on public data
(VitalDB, 5,151 surgeries), held to a strict honesty bar.

## Results (honest, on a locked held-out test set)

| What it does | Result |
|---|---|
| Warn before any deterioration | **AUPRC 0.454 / AUROC 0.849**, ~4.3 min early |
| Get the alarms right | **62–71% of alarms are real** (vs ~12% in deployed systems) |
| Say *which* of 9 events is coming | per-event **AUROC 0.74–0.96** |
| Work at another hospital, no retraining | **AUROC 0.78–0.80** (Seoul → UC-Irvine) |
| Does low blood pressure *cause* kidney injury? | mostly no — confounding (**RR 1.77 → 1.07**) |

## What's different
No existing system does all of this together — many events, cross-hospital transfer, honest cause analysis,
and trustworthy alarms. Held to a strict honesty bar: locked test split, a leaky model quarantined, every
failure reported — including that **hypotension is basically unpredictable** (shown 7 independent ways).
The honesty is the point.

**One practical upshot:** the two signals that matter most are **heart rate and SpO₂ (pulse-ox)** — both
cheap — so a low-cost version (ECG + pulse-ox only) could work where arterial lines aren't available.

## In this repo
- **`index.html`** — the live demo (the Pages site above). Replays real held-out cases; toggle **AI** to see the lead-time reveal.
- **`one-pager.html`** — one-page summary (open in a browser, then **Print → Save as PDF**).
- **`BENCHMARK_FINAL.md`** / **`benchmark_final.json`** — the full reproducible scoreboard.

## Honesty & data
Research prototype — not a medical device, diagnosis, or treatment recommendation. Trained and evaluated on
public **VitalDB**; external validation used UC-Irvine / MOVER under a data-use agreement (**that data is not
included here**). **No patient data is in this repository.**

---
*What I'm looking for: feedback, a research role, or a collaborator to take the strongest result
(cross-hospital transfer + the honest benchmark) toward a paper and outside validation. — Aastik Rajan, aastikc15@gmail.com*
