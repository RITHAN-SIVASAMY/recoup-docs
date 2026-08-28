# Recoup — Evaluation Protocol

**How every number in the submission is produced, and what would make it wrong.**

| Field | Detail |
|---|---|
| Version | 2.0 |
| Principle | *A number we cannot defend is worse than no number.* |

---

## 1. Why this document exists

The track bar asks for **measured** money recovered. Anyone can print a number. This document fixes the methodology *before* the results are known, so the results cannot be reverse-engineered into a better story. It is written to be handed to a sceptical reviewer.

Two commitments, made in advance:

1. **Pre-registration.** The primary metric, the batch size, the holdout rate, the significance threshold, and the analysis method are fixed here before any run. There is exactly one primary metric.
2. **Publish the null.** If the lift is not statistically significant, the dashboard, the README and the video say so, alongside the minimum detectable effect for the batch actually run. We report what we measured, not what we hoped.

---

## 2. Pre-registered analysis plan

| Parameter | Value |
|---|---|
| Primary metric | **Incremental recovered value (₹)** = (treated resolution rate − control resolution rate) × treated volume × mean recovered value |
| Unit of randomization | The `Case`, assigned at creation, immutable |
| Assignment | Seeded hash of `case_id` + salt, stratified by (root cause × amount band × segment) |
| Holdout rate | 20% at batch start, adaptive thereafter (§6) |
| Batch size | ≥500 cases (bar: 50) |
| Significance test | Two-proportion z-test on resolution rate |
| Alpha | 0.05, two-sided |
| Interval | 95% CI on the absolute lift |
| Variance reduction | CUPED with pre-period payment reliability as covariate; **the unadjusted estimate is always reported alongside** |
| Secondary metrics | Cost per ₹ recovered, contacts per recovered case, blocked-action count, uplift Qini, classifier macro-F1 and Brier, grounded-refusal rate |
| Exclusions from control | Cases above the merchant's value cap, and any case flagged legal-risk. Logged and counted in the report |

---

## 3. What each measurement means, exactly

### 3.1 Raw recovered (reported, but never the headline)

Sum of `recovered_amount` over treated cases that reached `recovered`. This is the number most submissions will present. We show it explicitly so the gap to the incremental number is visible, and label it:

> *Raw recovered ₹X. This overstates our impact, because some of these would have recovered without us. The honest number is below.*

### 3.2 Incremental recovered (the headline)

```
lift            = p_treated − p_control                     # resolution rates
incremental_₹   = lift × n_treated × mean_recovered_value
se              = sqrt( p_t(1−p_t)/n_t + p_c(1−p_c)/n_c )
z               = lift / se
CI95            = lift ± 1.96 × se
```

Reported as: `₹ incremental (95% CI: ₹low – ₹high), lift = X.X pp, p = 0.0XX, n_t / n_c`.

### 3.3 Minimum detectable effect

For the batch actually run, at 80% power and α=0.05, we report the smallest lift we *could* have detected. This is stated whether the result is significant or not, because it is the honest bound on what the experiment can say.

### 3.4 Cost of recovery

```
cost_of_recovery = Σ action_costs (treated) / incremental_₹
```

Reported in ₹ spent per ₹ incrementally recovered. Contacts avoided by the EV gate and the uplift segmentation are reported as **₹ saved by not acting** — the restraint metric.

---

## 4. Model evaluation

| Model | Metrics | Held-out protocol | CI gate |
|---|---|---|---|
| Root-cause classifier | Per-class precision/recall/F1, macro-F1, confusion matrix, **reliability curve + Brier score** | Stratified 70/15/15 split by case, no customer leakage across splits | macro-F1 ≥ 0.85, Brier ≤ 0.12 |
| Baseline / treated propensity | AUC-ROC, PR-AUC, calibration | Same split, arm-specific | AUC ≥ 0.70 |
| Uplift | **Qini coefficient**, uplift-at-k, per-decile lift, and agreement with the generator's ground-truth τ | Held-out cases only | Qini > 0 |
| PTP extraction | Precision, recall, F1 on 60+ labelled utterances; **false-positive rate reported separately** (the dangerous class) | Golden set, never trained on | FP rate ≤ 5% |
| Grounded Q&A | Citation accuracy, refusal rate on unanswerable questions, fabrication count | 40-question benchmark, 20 answerable / 20 not | Fabrications = 0, refusal ≥ 95% |
| Message safety | Prohibited-claims detection on generated copy | Adversarial prompt suite | 0 escapes |

**Leakage discipline:** the synthetic generator's ground-truth response curves are used *only* to validate the uplift model. They are never available as features at inference. A test asserts the ground-truth columns are absent from every feature matrix.

**Honesty requirement:** each model card contains a **"Known failure modes"** section written in plain English — the classes it confuses, the segments where it is unreliable, and what would fix it. A model card without that section fails review.

---

## 5. Policy and safety evaluation

| Property | How it is proven | Where |
|---|---|---|
| No action without a logged verdict | Architecture test scanning execution paths + property test over histories | `tests/property/test_authority.py` |
| Never exceed max contacts | Hypothesis over random histories | `tests/property/test_stopping.py` |
| Never contact after opt-out | Hypothesis | same |
| Never retry a revoked mandate | Hypothesis + explicit scenario | `tests/property/test_mandate.py` |
| Never act on a control case | Hypothesis + DB constraint | `tests/property/test_cohort.py` |
| Never execute one idempotency key twice | Hypothesis + unique index + Redis guard | `tests/property/test_idempotency.py` |
| Never message in quiet hours | Frozen-clock scenarios across timezone edges and DST | `tests/unit/test_quiet_hours.py` |
| Every executed action was staged first | Invariant test | `tests/property/test_staging.py` |
| Log is tamper-evident | Mutate a payload, assert the chain verification names the divergent event | `tests/unit/test_hash_chain.py` |
| State is reproducible | Replay equality: projection from events == stored projection | `tests/integration/test_replay.py` |

**Compliance rate** is not self-reported by the running system. It is computed by **replaying the audit log after the batch** and re-evaluating every executed action against the policy version recorded on it. Any action that would not be permitted counts as a violation. Target: zero.

---

## 6. The adaptive holdout, and why it is defensible

A permanent 20% control group is a permanent 20% tax on recovery, and a merchant will notice. Recoup treats the holdout as an evidence-gathering budget:

1. Start at 20%.
2. After each measurement window, run the sequential test with an alpha-spending function (O'Brien-Fleming style boundaries) so repeated looks do not inflate the false-positive rate.
3. Once the effect crosses the boundary at the configured confidence, decay the holdout toward a small monitoring slice (default floor 5%).
4. Continuously monitor: if the measured lift drifts outside its confidence band, the holdout automatically re-expands.

The full holdout schedule and every look are logged, so a reviewer can verify the sequence was not chosen after seeing the data.

**Anticipated challenge:** *"Isn't the control group costing your merchant money?"*
**Answer:** yes, and we quantify it — the report states the ₹ forgone by the holdout — but it is bounded, it decays automatically, and it buys the only number in the product that can be trusted. Without it, every other figure is a guess with a confident font.

---

## 7. Reproducibility

```bash
git clone --recurse-submodules https://github.com/<you>/recoup.git
cd recoup && make setup
make data SEED=42        # regenerates the identical 500-case batch (hash-checked)
make train               # retrains models, writes metrics + model cards
make demo                # runs the batch end-to-end, prints the headline block
make verify              # hash-chain verification + replay equality
make chaos               # runs the full chaos suite
```

Two runs from the same seed must produce byte-identical headline numbers. This is asserted in CI, not merely hoped for. Every reported figure in the README, the video and the dashboard is generated by `make demo` — no number in the submission is typed by hand.

---

## 8. Threats to validity, stated plainly

| Threat | Why it matters | Mitigation | Residual honesty |
|---|---|---|---|
| **Synthetic data** | Real distributions differ; performance will not transfer 1:1 | Generator, distributions and rationale published; metrics on held-out data only | We claim *"correct under the stated data-generating process"*, never *"X% in production"* |
| **Self-consistent simulator** | Uplift models and the response simulator share assumptions; the system could be scoring its own homework | Uplift validated against held-out ground truth; simulator parameters fixed before modelling | Reported as a known limitation in the README, the video and here |
| **Small batch** | Wide confidence intervals | Large batch, stratification, CUPED | MDE always reported; a null is published as a null |
| **Single merchant profile** | Generalization unknown | Multiple synthetic merchant profiles with different mixes | Cross-profile results reported, including where they disagree |
| **Model version drift during the build** | Results not comparable across days | Every decision records model and policy versions; the final run is a single clean batch | Only the final clean run is quoted |
| **Selective reporting** | The classic way demos lie | Pre-registration in this document, in git history, before results existed | The commit that fixed this protocol predates the results commit — verifiable |

---

## 9. The headline block

Every run prints exactly this, and it is what appears in the README and on screen in the video:

```
════════════════════════════════════════════════════════════════
RECOUP · BATCH b_2026_09_01 · seed 42 · 500 cases
────────────────────────────────────────────────────────────────
At risk                        ₹ 12,48,300
Raw recovered (treated)        ₹  3,91,200   ← overstates our impact
Incremental recovered          ₹  2,17,450   (95% CI ₹1,48,900 – ₹2,86,000)
Lift                           14.8 pp       z = 3.41, p = 0.0006
                               n_t = 400  n_c = 100  MDE = 8.9 pp
CUPED-adjusted                 ₹  2,24,100   (unadjusted shown above)
────────────────────────────────────────────────────────────────
Spend on contact               ₹      1,842
Cost per ₹ recovered           ₹     0.0085
₹ saved by not contacting      ₹     42,600  (sure things + sleeping dogs + EV floor)
────────────────────────────────────────────────────────────────
Actions blocked by policy      37   (quiet hours 14 · opt-out 9 · mandate 11 · cap 3)
Contacts per recovered case    1.8 median   ·  max touches respected: 100%
Cases in exception queue       6    (all triaged, none lost)
Audit chain                    VERIFIED · replay equality PASS
════════════════════════════════════════════════════════════════
```

Illustrative values. The real ones come from `make demo`, whatever they turn out to be.
