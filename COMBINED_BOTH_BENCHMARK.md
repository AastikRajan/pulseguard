# Co-trained (both-hospital) model — benchmark

**Date:** 2026-06-30 · `scripts/mover/train_combined_both.py` → `outputs/mover/combined_both_benchmark.json`
**Run:** full VitalDB (all 5,170 cases) + all MOVER/UC-Irvine SIS.

## Question
Aastik's request: train one model on **both** hospitals (VitalDB + MOVER/UC-Irvine) and benchmark it.

## Scope (honest)
The only task both datasets share is **hypoxemia (SpO₂<90 onset within 5 min), HR+SpO₂ only, 1-minute
resolution** (MOVER is 1-min and lacks VitalDB's 1-Hz / 35-channel richness). So a co-trained model is on
this **reduced shared task** — it is **NOT** the 35-channel 1-Hz unified EWS (AUPRC 0.454), which cannot be
co-trained because MOVER doesn't have the data. Different tasks; do not compare their AUPRC.

## Setup
- By-case 80/20 split, seed 2026, both hospitals. Full VitalDB (5,170 cases).
- Train windows: VitalDB 703,026 · MOVER 2,682,181. Test: VitalDB 176,850 (base 2.13%) · MOVER 665,827 (base 0.35%).
- Model: XGBoost (400 trees, depth 5), class-weighted; 12 features (6 trailing-window stats × HR, SpO₂).

## Results — AUROC / AUPRC on held-out patients

| Model trained on | VitalDB held-out | UC-Irvine held-out | Both pooled |
|---|---|---|---|
| VitalDB only | 0.864 / 0.272 | 0.834 / 0.034 *(zero-shot)* | 0.854 / 0.108 |
| UC-Irvine only | 0.793 / 0.132 *(zero-shot)* | 0.894 / 0.140 | 0.818 / 0.093 |
| **BOTH (co-trained)** | **0.850 / 0.234** | **0.890 / 0.118** | **0.900 / 0.173** |

(AUPRC differs across columns because hypoxemia base rate differs by site: 2.13% VitalDB vs 0.35% UC-Irvine — compare AUPRC only within a column.)

## Takeaways
1. **The co-trained model is the most balanced single model:** AUROC 0.850 on VitalDB and 0.890 on UC-Irvine, and **best on the combined population (0.900)**.
2. **Each single-hospital model is best on its own site but weaker on the other:** VitalDB-only 0.864→0.834 on UC-Irvine; UC-Irvine-only 0.894→0.793 on VitalDB. The co-trained model does not have that gap.
3. **Co-training trades a little for balance:** it costs ~0.014 AUROC on VitalDB's home turf (0.864→0.850) in exchange for +0.056 on UC-Irvine (0.834→0.890). Net: one model that holds up on both sites.
4. **Consistency check:** VitalDB-only zero-shot on UC-Irvine = 0.834 AUROC (more source data than the earlier 3k-case run lifted transfer from 0.79 to 0.83), consistent with the reported 0.78–0.83 cross-hospital range.

## Relationship to the headline benchmark
- **Full unified EWS (AUPRC 0.454 / AUROC 0.849):** VitalDB only, 35 channels @1Hz — stays VitalDB-trained + zero-shot external validation (MOVER can't co-train it).
- **Shared hypoxemia task (this doc):** a genuine both-hospital model that holds up on both sites — the answer to "train it on both."
