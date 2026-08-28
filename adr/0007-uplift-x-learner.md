# ADR-0007 · X-learner for uplift estimation

**Status:** Accepted · Phase 03

## Context
Targeting requires τ = P(recover | treated) − P(recover | untreated) per case. Options: two-model (T-learner), X-learner, causal forest, or uplift trees.

## Decision
X-learner over two LightGBM base models. `causalml` is an optional dependency for cross-checking, not a requirement.

## Consequences
- **Positive:** transparent and debuggable; trains in seconds so it fits the CI gate; performs better than a naive T-learner under imbalanced arms, which is exactly our situation with a 20% holdout.
- **Negative:** less robust than a causal forest on complex heterogeneity; sensitive to base-model calibration. Mitigated by calibrating both base models and by evaluating with a Qini curve against the generator's ground-truth τ.
- **Fallback:** if uplift is unstable, ship a documented heuristic (treated propensity minus self-heal prior), labelled as a heuristic in the UI. Never present a heuristic as a model.
