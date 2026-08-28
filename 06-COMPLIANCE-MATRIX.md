# Recoup — Compliance and Governance Matrix

**Every rule, mapped to the config that holds it, the code that enforces it, and the test that proves it.**

| Field | Detail |
|---|---|
| Version | 2.0 |
| Standing caveat | Recoup claims **alignment by design**, not legal certification. Every regulatory constant below is a named, configurable value with an owner. Before any production use, each must be re-verified against the currently effective NPCI, RBI, TRAI and MeitY/DPDP instruments. This document is the artifact that makes that re-verification a ten-minute task instead of an archaeology project. |

---

## 1. Why this document exists

Most submissions mention compliance in a README bullet. A fintech reviewer's actual question is narrower and harder: *"show me the line of code, and show me the test."* This matrix answers that question for every rule, and it is the reason the regulatory layer is kept **separate from merchant-tunable business policy** — so that a merchant loosening their own escalation settings can never loosen a legal constraint.

---

## 2. Layer separation

| Layer | Who may change it | Where it lives | Example |
|---|---|---|---|
| **Regulatory guardrails** | Compliance owner only; changes require an ADR | `policies/regulatory.yaml` (separate file, separate review) | Quiet hours, mandate retry caps, consent requirements |
| **Merchant business policy** | Merchant, via settings | `policies/merchant/<id>.yaml` | Approval threshold, EV floor, contact caps, ladder aggressiveness |
| **Learned behaviour** | The system, online | Bandit posteriors, model artifacts | Channel and send-time choice — **only within the two layers above** |

A merchant can make Recoup gentler. A merchant cannot make it non-compliant. The bandit can explore. It cannot escape.

---

## 3. The matrix

### 3.1 Communication and consent

| ID | Rule (as implemented) | Basis | Config key | Enforced in | Proven by |
|---|---|---|---|---|---|
| REG-COMM-01 | No commercial contact outside permitted hours (default 09:00–20:00 IST, merchant timezone aware) | TRAI UCC/DND framework — *verify current preference categories and time bands* | `regulatory.quiet_hours` | `policy/rules/quiet_hours.py` | `tests/unit/test_quiet_hours.py` (incl. DST and timezone-edge cases) |
| REG-COMM-02 | Contact requires a recorded consent basis per channel; absence of consent is a DENY, not a default-allow | DPDP Act 2023 — consent and notice | `regulatory.consent.required_channels` | `policy/rules/consent.py` | `tests/property/test_consent.py` |
| REG-COMM-03 | Opt-out is honoured immediately, permanently, and across **all** cases for that customer | TRAI + DPDP withdrawal of consent | `regulatory.opt_out.propagate_all_cases: true` | `policy/rules/opt_out.py` | `tests/property/test_stopping.py::test_no_contact_after_optout` |
| REG-COMM-04 | Recovery messages are transactional/service-category; promotional content is a separate, non-recovery template class | TRAI template/header registration regime | `templates/*.yaml: category` | `execution/templates.py` | `tests/unit/test_template_category.py` |
| REG-COMM-05 | Every outbound message carries sender identification and an opt-out affordance | TRAI + general fair-practice | template contract | `execution/renderer.py` | `tests/unit/test_message_contract.py` |
| REG-COMM-06 | Per-customer rolling contact cap across all cases (default 3 / 30 days) | Merchant policy hardened into a guardrail because per-case correctness is not aggregate correctness | `regulatory.contact_fatigue` | `economics/fatigue.py` | `tests/property/test_fatigue_budget.py` |

### 3.2 Recurring payments and mandates

| ID | Rule (as implemented) | Basis | Config key | Enforced in | Proven by |
|---|---|---|---|---|---|
| REG-MAND-01 | `mandate_revoked` and `mandate_insufficient_balance` are **never** silently retried | NPCI UPI Autopay / RBI e-mandate framework — *verify current cadence rules* | `regulatory.mandate_retry.never_retry_causes` | `policy/rules/mandate.py` | `tests/property/test_mandate.py::test_never_retry_revoked` |
| REG-MAND-02 | Retries for `mandate_technical_failure` obey a max-per-cycle cap and a minimum gap | NPCI retry cadence | `regulatory.mandate_retry.{max_per_cycle,min_gap}` | `policy/rules/mandate.py` | `tests/unit/test_mandate_cadence.py` |
| REG-MAND-03 | Where a recurring debit requires advance customer notification, Recoup verifies the notification exists before proposing a retry; otherwise it routes to notify-then-retry | RBI e-mandate pre-debit notification | `regulatory.mandate.pre_debit_notice_required` | `policy/rules/mandate.py` | `tests/unit/test_pre_debit_notice.py` |
| REG-MAND-04 | Any mandate state requiring customer authentication produces a re-authorization link, never an automated debit attempt | RBI AFA requirements for e-mandates | ladder `mandate_revoked` | `policies/ladders.yaml` + `policy/evaluator.py` | `tests/property/test_mandate.py::test_reauth_only` |
| REG-MAND-05 | Recurring-debit value thresholds that trigger additional authentication are represented as a named constant, not an inline number | RBI e-mandate AFA threshold — *verify current value* | `regulatory.mandate.afa_threshold_inr` | `policy/rules/mandate.py` | `tests/unit/test_afa_threshold.py` |

### 3.3 Money movement and authorization

| ID | Rule | Basis | Config key | Enforced in | Proven by |
|---|---|---|---|---|---|
| GOV-MONEY-01 | No money-moving or contact action executes without a logged policy verdict | Internal invariant; the basis of auditability | — | `execution/dispatcher.py` | `tests/property/test_authority.py` (architecture + property) |
| GOV-MONEY-02 | Every action carries a deterministic idempotency key; duplicates are suppressed and logged | Payment-systems hygiene | — | `execution/idempotency.py` + unique index | `tests/property/test_idempotency.py` |
| GOV-MONEY-03 | Actions above the value/risk threshold require human approval before execution | Merchant governance | `merchant.approval.value_threshold_inr` | `policy/rules/approval.py` | `tests/integration/test_approval_gate.py` |
| GOV-MONEY-04 | Every executed action was `staged` first and was cancellable for its undo window | Reversibility requirement (FR-7.2) | `merchant.staging.undo_window` | `execution/staging.py` | `tests/property/test_staging.py` |
| GOV-MONEY-05 | Merchant exposure cap forces queueing rather than auto-execution once exceeded | Merchant governance | `merchant.exposure_cap_inr` | `policy/rules/exposure.py` | `tests/integration/test_exposure_cap.py` |
| GOV-MONEY-06 | A global kill switch halts all autonomous action and cancels in-flight staged actions | Operational safety | `merchant.kill_switch` | `execution/killswitch.py` | `tests/integration/test_kill_switch.py` |
| GOV-MONEY-07 | Formal-notice-class actions are drafted but never sent autonomously | Fair-practice / legal risk | ladder `receivable_overdue` | `policies/ladders.yaml` | `tests/property/test_b2b_ladder.py` |

### 3.4 Data protection and security

| ID | Rule | Basis | Config key | Enforced in | Proven by |
|---|---|---|---|---|---|
| SEC-DATA-01 | No raw card PAN enters the system at any point; tokenized references only | PCI-DSS scope avoidance by design | — | ingestion normalizers | `tests/unit/test_no_pan.py` (regex assertion over all persisted payloads) |
| SEC-DATA-02 | PII is pseudonymized before any LLM call | DPDP Act 2023 — purpose limitation and minimization | `llm.redaction.enabled: true` (not switchable in prod) | `llm/redaction.py` | `tests/unit/test_prompt_redaction.py` (asserts no phone/email/name pattern can reach an outbound prompt) |
| SEC-DATA-03 | Webhook signatures verified; unsigned or mismatched payloads never create a case | Integration security | `razorpay.webhook_secret` | `ingestion/webhook.py` | `tests/unit/test_webhook_signature.py` |
| SEC-DATA-04 | Recovery links are HMAC-signed, single-use, expiring, non-enumerable, rate-limited | Customer safety, phishing resistance | `links.{ttl,secret}` | `execution/links.py` | `tests/unit/test_recovery_links.py` |
| SEC-DATA-05 | Customer-level data export and deletion supported; retention configurable | DPDP Act 2023 — data principal rights | `privacy.retention_days` | `api/privacy.py` | `tests/integration/test_privacy_rights.py` |
| SEC-DATA-06 | No secrets in the repository; pre-commit secret scanning | Basic hygiene | — | `.pre-commit-config.yaml` (gitleaks) | CI job `secret-scan` |

### 3.5 Conduct

| ID | Rule | Basis | Enforced in | Proven by |
|---|---|---|---|---|
| CON-01 | Recoup is recovery-only and strictly defensive. No pressure tactics, no manufactured urgency, no false consequence, no shaming, no third-party disclosure | Fair-practice norms; Razorpay's defensive-agent guidance | `llm/safety/prohibited_claims.py` applied to every generated message | `tests/llm_eval/test_message_safety.py` (adversarial suite) |
| CON-02 | Voice calls disclose that the caller is an automated assistant, identify the merchant, and offer opt-out and human transfer, honoured in-call | Fair-practice / consent | `voice/graph.py` mandatory nodes | `tests/unit/test_voice_graph.py::test_disclosure_is_unskippable` |
| CON-03 | Distress, dispute or legal keywords terminate the flow and raise a human exception | Duty of care | `voice/guards.py` | `tests/unit/test_voice_guards.py` |
| CON-04 | The grounded Q&A layer never speculates; absence of a logged fact produces a refusal naming what would need to be logged | Auditability | `audit/qa.py` | `tests/llm_eval/test_grounded_qa.py` (20 unanswerable questions) |
| CON-05 | High relationship-risk accounts cap escalation and route to a human rather than escalating harder | Customer-relationship protection | `policy/rules/relationship_brake.py` | `tests/unit/test_relationship_brake.py` |

---

## 4. Governance operations

| Control | Mechanism |
|---|---|
| **Change review** | Regulatory config changes require an ADR and a CI-visible diff; the policy content hash changes, so every subsequent decision records the new version |
| **Attribution** | Every event records actor (`system` / `model` / `human:<id>` / `provider`), timestamp, policy version and model versions |
| **Post-hoc compliance audit** | `make verify-compliance` replays the batch log and re-evaluates every executed action against the policy recorded on it; any action that would not be permitted is reported as a violation |
| **Tamper evidence** | Hash-chained event log; `make verify` names the first divergent event |
| **Incident handling** | Anything the system could not resolve lands in the exception queue with a truthful reason; nothing is silently dropped |
| **Kill switch drill** | Exercised in the demo, not just implemented |

---

## 5. Open items and honest limitations

1. **Statutory values are defaults, not legal advice.** Quiet-hours bands, mandate retry caps, AFA thresholds and notification windows are conservative placeholders pending verification against current circulars. Each is a single named constant with a citation comment and an owner.
2. **DND registry integration is simulated.** Production requires DLT/registry lookups through a registered provider; the interface exists and is stubbed, and the stub is labelled as a stub in the UI.
3. **Consent provenance is synthetic.** Real deployments must capture consent artifacts at collection time; the schema supports it, the demo data fabricates it.
4. **Single-tenant demo.** Multi-merchant isolation, RBAC and SSO are designed for but not implemented in v1.
5. **No legal review has been performed.** This is a buildathon artifact operating on test-mode APIs and synthetic data, and it says so everywhere it could be mistaken for otherwise.
