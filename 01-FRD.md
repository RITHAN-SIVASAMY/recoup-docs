# Recoup — Functional Requirements Document

**A risk-aware, causally-measured revenue recovery agent for Razorpay merchants**

| Field | Detail |
|---|---|
| Document | Recoup — Functional Requirements Document |
| Version | **2.0** (supersedes v1.0, August 2026) |
| Track | Razorpay AI Buildathon — Track 03: AI Revenue Recovery |
| Track brief | *"Find revenue that's slipping away and win it back. Build an agent that detects revenue at risk, determines the right intervention, and executes a bounded recovery workflow: from payment failures and checkout abandonment to overdue receivables."* |
| Track bar | *"Don't just identify the problem. Show measured money recovered across a batch, with compliant escalation, stopping rules, and an audit trail."* |
| Status | Approved for build |
| Related docs | `02-PROBLEM-AND-DIFFERENTIATION.md`, `03-ARCHITECTURE.md`, `04-EXECUTION-PLAN.md`, `05-EVALUATION-PROTOCOL.md`, `06-COMPLIANCE-MATRIX.md` |
| Change log | v2.0 adds: uplift-based targeting (FR-3), an economic decision layer (FR-4), reversibility and undo (FR-7), a constrained contextual bandit (FR-9), a self-serve recovery microsite (FR-12), adaptive holdout + CUPED (FR-13), event-sourced replay (FR-14), a what-if simulator (FR-15), and a chaos/resilience harness (FR-16). Regulatory requirements are separated from business policy (FR-6). |

---

## 1. Executive summary

Recoup is an autonomous, risk-aware revenue recovery agent for Razorpay merchants. It watches the four places a merchant's revenue quietly leaks — failed one-time payments, abandoned checkouts, failed UPI Autopay / e-mandate renewals, and overdue B2B receivables — diagnoses *why* the money didn't come in, decides **whether it is worth chasing at all**, chooses the intervention with the highest *incremental* payoff, and executes a bounded, compliant recovery workflow across SMS, WhatsApp, email, a self-serve recovery page, and a Hinglish voice call. Every action is logged, replayable, explainable, and reversible before it becomes irreversible.

By the time this event runs, every serious team will have used an AI coding agent to scaffold a working stack. Execution speed is the baseline, not the differentiator. Recoup's differentiation is deliberately placed in five properties that a generic prompt never produces:

1. **It measures causally, not descriptively.** Recovered revenue is reported as incremental lift against a held-out control cohort with a significance test and a confidence interval — not a raw count of retries that later succeeded. The holdout is *adaptive*: it shrinks and switches off once the effect is established, so honesty does not cost the merchant money indefinitely (FR-13).
2. **It targets uplift, not propensity.** The model that matters is not "will this payment recover?" but "will this payment recover *because we acted*?" Cases that would have self-healed are deliberately left alone. This is the difference between a busy bot and a profitable one (FR-3).
3. **It knows what an action costs.** Every contact has a price (SMS, WhatsApp template, voice minute, human review time). Recoup will not spend ₹4 of contact and goodwill chasing ₹60 with an 8% uplift. Expected value gates escalation before policy even gets a say (FR-4).
4. **It cannot become a spam engine or double-charge anyone.** Every money-moving or contact-initiating action passes a deterministic policy engine — regulatory guardrails, hard stopping rules, exposure caps, idempotency keys, a human-approval gate above a configurable threshold, a staged undo window, and a global kill switch. The LLM proposes; deterministic code disposes (FR-5, FR-6, FR-7).
5. **It reasons about *who* it is chasing, not just *what* failed.** A UPI Autopay technical decline, an overdue SME invoice from a five-year account, and a stranger's expired card are governed by different rules and different relationship economics, by design rather than by accident (FR-8, FR-11).

The system is buildable inside an 8-day window on Razorpay test-mode APIs and synthetic data, but every requirement below is written so that it would still be correct in a production deployment. This is an FRD, not a pitch deck.

**Headline claim we intend to defend on stage:** *"Across a 500-case synthetic batch, Recoup recovered ₹X — and here is the control group, the z-test, and the audit trail that proves ₹X is the amount we caused, not the amount that happened."*

---

## 2. Problem statement and market gap

### 2.1 The problem

Revenue loss for a digital-first merchant rarely arrives as one clean, single-cause event. A payment fails at the bank; a customer abandons checkout mid-flow; a UPI Autopay renewal silently stops working; an invoice quietly ages past due. Each has a different root cause, a different correct response, and a different regulatory envelope. Most merchants — and essentially every hackathon recovery bot — collapse all four into a single bucket called *"failed to collect"* and answer it with one blunt instrument: retry, email, stop.

Three structural facts make this worse in India than almost anywhere else:

- **Rail fragmentation.** UPI, cards, net-banking, and wallets fail for different reasons and recover through different mechanisms. A retry-everything strategy underperforms here more than in single-rail markets.
- **Recurring payments are regulated, not just technical.** UPI Autopay and card e-mandates sit inside a framework of pre-debit notification, additional-factor authentication thresholds, and retry cadence limits. A bot that blind-retries a revoked mandate is not merely ineffective; it is a compliance incident.
- **Contact is regulated too.** Commercial communication in India is governed by TRAI's UCC/DND framework and consent registration; personal data by the DPDP Act, 2023. "Fire an SMS on a timer" is an operational liability for a real merchant.

### 2.2 Why this is a real gap, not an invented one

Funded companies exist to plug individual pieces of this (Chargebee Retention, Recurly, Butter Payments, ProsperStack, Gaviti and Growfin on the receivables side). Their existence validates the market. What they largely optimize is *retry timing and dunning sequence*. None of them, at the SME tier an Indian merchant can actually afford, combine: root-cause-specific remediation **+** uplift-based targeting **+** relationship-aware escalation **+** causally measured impact **+** a reviewable audit trail. That intersection is the gap Recoup targets.

### 2.3 Why "measured" is the load-bearing word in the brief

The track bar does not ask for a bot that reduces failures. It asks for **measured money recovered across a batch, with compliant escalation, stopping rules, and an audit trail.** That is a deliberate filter. The expected failure mode of the submission pool is a large, uncontrolled, inflated recovery number. The differentiator is being the team whose number survives the question *"how do you know they wouldn't have paid anyway?"*

---

## 3. Goals, non-goals and success metrics

### 3.1 Goals

- **G1** Detect revenue-at-risk events across four sources: failed one-time payments, abandoned checkouts, failed UPI Autopay / e-mandate renewals, overdue B2B receivables.
- **G2** Diagnose root cause with a trained, calibrated, explainable model — not a hardcoded lookup table.
- **G3** Decide *whether* to act using uplift and expected value, not raw propensity; leave self-healing and hopeless cases alone.
- **G4** Execute a bounded, multi-channel, regulator-aware recovery workflow with hard stopping rules and reversibility.
- **G5** Prove impact with an incremental (treatment-vs-control) recovered-revenue figure with a significance test.
- **G6** Produce a complete, tamper-evident, queryable audit trail from which any case can be fully reconstructed and any decision explained without speculation.
- **G7** Fail gracefully: no case is ever silently dropped; every case terminates in `recovered`, `stopped_by_policy`, `abandoned_uneconomic`, `exception`, or `control_untouched`.

### 3.2 Non-goals for v1

- Recoup does not move real money. It runs entirely on Razorpay **test-mode** APIs and synthetic data.
- Recoup does not do fraud detection, dispute handling, or chargeback defence (Track 02's problem). It assumes the transaction is legitimate and simply did not complete.
- Recoup does not autonomously execute legal or formal-collections escalation. It stops at a human-approval gate and drafts, never sends, anything resembling a formal notice.
- Recoup does not negotiate. Voice and chat are bounded flows, not open-ended settlement discussions.
- Recoup is not a general-purpose messaging platform; it owns recovery communication only.

### 3.3 Success metrics (mapped to the judging bar)

| # | Metric | Definition | Target for the demo batch | Which part of the bar it answers |
|---|---|---|---|---|
| M1 | **Incremental recovery** | (treated resolution rate − control resolution rate) × treated volume × average value, in ₹ and % | Reported with 95% CI and two-proportion z-test, whatever the value | "measured money recovered" |
| M2 | Raw vs. incremental gap | Raw recovered ₹ shown alongside incremental ₹ | Both shown, gap explained on the dashboard | Honesty of metrics |
| M3 | Root-cause classification quality | Macro-F1 and per-class precision/recall on a held-out test set + calibration (Brier score, reliability curve) | Macro-F1 ≥ 0.85, Brier ≤ 0.12, failure modes named | "meaningful use of AI" |
| M4 | Uplift quality | Qini coefficient / uplift-at-k on held-out data | Positive Qini; top-3 deciles carry the lift | Targeting, not spraying |
| M5 | Policy compliance rate | % of executed actions that passed every guardrail check, verified by replaying the audit log | **100%**, with ≥1 deliberately blocked action demonstrated | "compliant escalation, stopping rules" |
| M6 | Contact restraint | % of contacted customers within policy-max touches; median touches per recovered case | 100% within max; median ≤ 2 | Not a spam engine |
| M7 | Cost of recovery | ₹ spent on contact per ₹ incrementally recovered | Reported; EV-negative cases visibly skipped | Practical merchant impact |
| M8 | Audit completeness | % of actions whose full decision trail replays deterministically from the event log | **100%** | "audit trail" |
| M9 | Grounded-answer integrity | On a 40-question eval set (20 answerable, 20 unanswerable), correct-citation rate and refusal rate | ≥95% refusal on unanswerable; zero fabricated citations | Explainability |
| M10 | Resilience | Chaos suite: duplicate, out-of-order, malformed, and mid-flight-crash events produce zero duplicate actions and zero lost cases | 100% pass | "graceful failure handling" |

---

## 4. Stakeholders, personas and jobs-to-be-done

| Persona | Job to be done | Surface they touch | What makes them adopt |
|---|---|---|---|
| **Merchant finance / ops lead** (primary buyer) | "Get my leaked revenue back without burning customers or breaking rules — and show me it actually worked." | Dashboard, approval queue, what-if simulator, audit explorer, exportable report | The incremental number and the approval queue. They can *trust* it because they can *stop* it. |
| **Ops executive** (daily user) | "Tell me the 20 cases worth my time today and why." | Work queue ranked by expected incremental value, exception queue | Prioritized queue with a one-line reason and a one-click action |
| **Customer / payer** | "Let me fix this payment in one tap and stop messaging me." | SMS / WhatsApp / email / voice, self-serve recovery page, one-tap opt-out | A single-use link that just works; never contacted twice about the same thing |
| **Compliance reviewer** | "Prove every contact respected consent, DND, and mandate rules." | Compliance matrix view, audit trail, policy version diff | Policy-as-code with version history and a replayable log |
| **Razorpay platform** | Source of truth for payments, mandates, checkout, and links | Test-mode webhooks + REST APIs | Signature-verified, idempotent, back-pressure-safe integration |

---

## 5. System overview and high-level architecture

Recoup is six cooperating layers. Detail lives in `03-ARCHITECTURE.md`; this is the requirement-facing summary.

| Layer | Responsibility | Key components |
|---|---|---|
| **1. Ingestion** | Capture revenue-at-risk events reliably and exactly once | Signature-verified Razorpay webhook receiver, checkout-abandonment tracker, mandate-status poller, receivables importer, dedupe + dead-letter queue |
| **2. Understanding** | Turn a raw event into a diagnosed, scored, prioritized case | Root-cause classifier (calibrated, SHAP), recovery-propensity model, **uplift model**, LTV / relationship scorer, trust score |
| **3. Economics** | Decide whether acting is worth it, and which action pays best | Expected-value engine, channel cost model, contact-fatigue budget, EV-gated action ranking |
| **4. Decision & policy** | Turn a proposed action into a *permitted* action | Policy-as-code engine, escalation ladders, regulatory guardrails, exposure cap, human-approval gate, staged-action undo buffer, idempotency guard, kill switch |
| **5. Execution** | Carry out permitted actions across channels | Channel adapters (SMS / WhatsApp / email / voice) with a deterministic simulator, self-serve recovery microsite, Razorpay payment-link and mandate re-auth flows, PTP capture |
| **6. Measurement & trust** | Prove impact and make every decision explainable | Cohort assigner, adaptive holdout controller, incremental-recovery calculator, event-sourced hash-chained audit log, deterministic replay, grounded Q&A |

**Where the differentiation lives:** layers 2, 3, 4 and 6. Layers 1 and 5 are plumbing that any code-generation agent scaffolds in an hour, which is exactly why this document spends no detail there.

**The authority model, stated once:** the LLM never executes. It may *propose* an action, *draft* a message, *extract* a promise, or *explain* a log. Every state transition that touches money, contact, or a customer record is executed by deterministic, unit-tested Python behind the policy engine. This is the single most important architectural sentence in the document.

---

## 6. Functional requirements

Sixteen modules, ordered by pipeline stage. Each opens with the gap it closes, because the standing instruction for this build is that every feature must be both **essential** and **non-generic**.

---

### FR-1 Event ingestion and normalization

> **Gap closed:** most recovery bots treat every failure identically. In a fragmented payment stack, a soft bank decline, an OTP timeout, and a revoked mandate need entirely different responses; collapsing them is the single biggest reason generic retry bots underperform.

| ID | Requirement | Detail |
|---|---|---|
| FR-1.1 | Ingest failed one-time payments | Consume Razorpay test-mode `payment.failed` webhooks: decline code/description, amount, method, order and customer reference |
| FR-1.2 | Ingest checkout abandonment | Track `order.paid`-less orders and initiated-but-uncompleted checkout sessions past a configurable idle threshold (default 30 min) |
| FR-1.3 | Ingest mandate / Autopay failures | Subscribe to `subscription.*` / e-mandate status changes; treat as a distinct source, never as a generic payment failure |
| FR-1.4 | Ingest overdue receivables | Import a synthetic B2B invoice batch (CSV + API) with due date, amount, terms, and account relationship metadata |
| FR-1.5 | Normalize into one Case | All four sources map to one internal `Case` schema (§8) so every downstream module is source-agnostic |
| FR-1.6 | Webhook authenticity | Verify Razorpay webhook signatures; reject and log unsigned or mismatched payloads without creating a case |
| FR-1.7 | Exactly-once ingestion | Deduplicate on `(source, provider_event_id)`; a replayed or duplicated webhook must produce **zero** additional cases or actions |
| FR-1.8 | Out-of-order and late events | Events are applied by provider timestamp with a reconciliation pass; a late-arriving success event must retire an in-flight recovery attempt |
| FR-1.9 | Dead-letter queue | Any event that cannot be parsed, verified, or normalized lands in a DLQ with the raw payload and is surfaced in the exception queue — never dropped |

---

### FR-2 Root-cause classification

> **Gap closed:** "payment failed" is not a diagnosis. The correct next step for an OTP timeout, an insufficient-funds decline, and a revoked mandate share nothing but the word "failed".

| ID | Requirement | Detail |
|---|---|---|
| FR-2.1 | Decline taxonomy | Classify into: `bank_soft_decline`, `insufficient_funds`, `otp_timeout_or_auth_abandon`, `network_or_gateway_error`, `issuer_risk_block`, `card_expired_or_invalid`, `mandate_revoked`, `mandate_insufficient_balance`, `mandate_technical_failure`, `checkout_abandonment`, `receivable_overdue`, `unknown` |
| FR-2.2 | Trained model, not a lookup | Gradient-boosted multi-class classifier (LightGBM/XGBoost) over decline code, method, issuer/bank, amount band, time-of-day, device, customer history, prior-attempt features |
| FR-2.3 | Held-out evaluation | Per-class precision/recall/F1 on a held-out test set, reported honestly including the classes it gets wrong; a confusion matrix ships in the repo |
| FR-2.4 | **Calibrated confidence** | Probabilities calibrated (isotonic or Platt) and validated with a reliability curve + Brier score, because downstream EV maths consumes these numbers as real probabilities |
| FR-2.5 | Attribution | SHAP top-k contributing features stored on the case and surfaced in the audit trail and the UI |
| FR-2.6 | Low-confidence routing | Below a configurable confidence floor the case is classified `unknown`, routed to the conservative ladder, and flagged for human review — never guessed into an aggressive path |
| FR-2.7 | Cold start | With no customer history, the model falls back to documented priors per decline code; the case is marked `cold_start` so its metrics can be reported separately |

---

### FR-3 Recovery propensity **and uplift** scoring

> **Gap closed:** propensity answers "will this recover?"; uplift answers "will this recover *because of us*?" A bot optimizing propensity spends its budget on cases that would have self-healed and takes credit for them. This is the model-level twin of the control group.

| ID | Requirement | Detail |
|---|---|---|
| FR-3.1 | Baseline propensity | P(recover \| case, no intervention) — the self-heal probability |
| FR-3.2 | Treated propensity | P(recover \| case, intervention, channel, timing) |
| FR-3.3 | **Uplift score** | τ(case, action) = FR-3.2 − FR-3.1, estimated with a two-model / X-learner approach trained on the accumulated treatment-vs-control history (bootstrapped from the synthetic generator's ground-truth response curves) |
| FR-3.4 | Segment behaviour | Cases are bucketed into `sure_thing` (recovers anyway), `persuadable` (positive uplift), `lost_cause` (no response), `sleeping_dog` (negative uplift — contact makes it worse, e.g. a cancellation-prone subscriber). Only **persuadables** are worth contacting |
| FR-3.5 | Uplift evaluation | Qini curve and uplift-at-k reported on held-out data; the dashboard shows the ₹ saved by *not* contacting sure things and sleeping dogs |
| FR-3.6 | Continuous re-scoring | Scores recompute after every inbound signal (open, click, reply, promise, partial payment); scores are versioned so an old decision can be explained with the score it actually used |
| FR-3.7 | Priority score | `priority = uplift × amount_at_risk × urgency_decay × relationship_weight` ranks the queue (relationship weight from FR-8) |

---

### FR-4 Economic decision layer

> **Gap closed:** almost no recovery bot knows what its own actions cost. Spending ₹3.20 of WhatsApp, voice minutes and goodwill to chase ₹90 at 6% uplift destroys value while reporting a success. Recoup refuses to take actions with negative expected value and says so on the case.

| ID | Requirement | Detail |
|---|---|---|
| FR-4.1 | Channel cost model | Per-action cost in ₹ configured per channel (SMS, WhatsApp template, email, voice minute, human-review minute), plus a configurable **goodwill cost** per contact that grows with contact count |
| FR-4.2 | Expected value of an action | `EV(action) = uplift(action) × amount_at_risk × merchant_margin − cost(action) − goodwill_cost(contact_n)` |
| FR-4.3 | **EV gate** | An action whose EV is below a configurable floor is never proposed; if no action clears the floor, the case terminates as `abandoned_uneconomic` with the arithmetic recorded |
| FR-4.4 | Action selection | Among permitted actions, choose the highest-EV one; ties break toward the lower-intrusiveness channel |
| FR-4.5 | Contact-fatigue budget | A per-customer rolling budget (default: max 3 recovery contacts per 30 days across **all** cases) that no individual case can exceed, protecting the customer from being correct-per-case and spammed in aggregate |
| FR-4.6 | Merchant-level budget | Optional daily spend and daily contact-volume caps; exceeding either routes remaining cases to the queue instead of executing |
| FR-4.7 | Reporting | Cost of recovery (₹ spent per ₹ incrementally recovered) is a first-class dashboard metric, not an afterthought |

---

### FR-5 Bounded escalation policy engine (policy-as-code)

> **Gap closed:** "compliant escalation, stopping rules" is explicit in the brief. Most teams will hardcode *retry 3 times then stop*. A real merchant needs rules that differ by failure type, are versioned, are testable, and cannot be silently violated — including by an LLM.

| ID | Requirement | Detail |
|---|---|---|
| FR-5.1 | Policies are code | Escalation ladders and rules are declared in versioned, human-readable YAML in the repo, hot-reloadable, diffable in git, and unit-tested. No policy lives in a prompt |
| FR-5.2 | Per-cause ladders | e.g. `bank_soft_decline` → immediate silent retry → SMS + link; `insufficient_funds` → wait 24–48h → nudge at a salary-cycle-aware hour; `mandate_revoked` → **never retry**, send re-authorization link only; `receivable_overdue` → soft reminder → firm reminder → *human-approved* formal notice |
| FR-5.3 | Hard stopping rules | Max attempts per case, max contacts per channel, mandatory cooldown windows, immediate stop on opt-out, immediate stop on payment success, immediate stop on dispute/refund signal |
| FR-5.4 | Deterministic evaluation | The policy engine is pure, side-effect-free, and returns `ALLOW / DENY / REQUIRE_APPROVAL` with a machine-readable reason code and the exact rule ID that fired |
| FR-5.5 | **Invariant tests** | Property-based tests assert system-wide invariants over randomly generated case histories: never exceed max attempts, never contact after opt-out, never retry a revoked mandate, never act on a control case, never act twice on one idempotency key |
| FR-5.6 | Policy simulator | Any policy change can be replayed against the historical case log to show what *would* have changed, before it is adopted |
| FR-5.7 | Versioned decisions | Every decision records the policy version hash that produced it, so a historical action is explained by the rules in force at the time |

---

### FR-6 Regulatory guardrails

> **Gap closed:** these are separated from business policy on purpose. Business rules are tunable by the merchant; regulatory rules are not. Mixing them is how a configuration change becomes a violation.

| ID | Requirement | Detail |
|---|---|---|
| FR-6.1 | Consent and opt-out | Contact requires a recorded consent basis per channel; opt-out is honoured immediately, permanently, and across all cases for that customer |
| FR-6.2 | DND / quiet hours | No commercial contact outside configured permitted hours (default 09:00–20:00 IST) or to a DND-registered number on a non-transactional template; enforced at the policy layer, not in the message template |
| FR-6.3 | Template discipline | Promotional and transactional message templates are distinct entities with distinct permissions; recovery messages are transactional/service-category and must not carry marketing content |
| FR-6.4 | Mandate retry cadence | UPI Autopay / e-mandate retries follow a configured cadence with a hard cap per debit cycle, and **must distinguish** `mandate_technical_failure` (retryable) from `mandate_revoked` / `mandate_insufficient_balance` (not silently retryable) |
| FR-6.5 | Pre-debit notification awareness | Where a recurring debit requires advance notification to the customer, Recoup checks that the notification exists before proposing a retry and otherwise routes to a notify-then-retry path |
| FR-6.6 | Re-authorization path | Any mandate state requiring customer authentication produces a re-authorization link, never an automated debit attempt |
| FR-6.7 | Data protection | Tokenized customer references only; no card PAN ever enters the system (PCI-out-of-scope by design); PII is redacted before any prompt reaches an LLM; deletion and export requests are supported at the customer level |
| FR-6.8 | Configurable, cited, and reviewable | Every regulatory parameter is a named config value with a citation comment and an owner, and appears in `06-COMPLIANCE-MATRIX.md` mapped to the test that enforces it. Defaults are conservative; the document states plainly that exact statutory values must be re-verified against current NPCI/RBI/TRAI circulars before any production use |

---

### FR-7 Human authority, reversibility and kill switch

> **Gap closed:** the executive summary of every agent project claims actions are "reversible". Almost none specify the mechanism. This one does.

| ID | Requirement | Detail |
|---|---|---|
| FR-7.1 | Approval gate | Any action above a configurable value/risk threshold, any action on a flagged account, and every formal-notice-class action requires explicit human sign-off before execution |
| FR-7.2 | **Staged actions and undo window** | Approved and auto-approved actions enter a `staged` state with a configurable hold (default 60s for contact, 5 min for money-moving) during which they can be cancelled with one click. Nothing leaves the system without passing through a cancellable state |
| FR-7.3 | Exposure cap | A merchant-level cap on total value under active recovery; exceeding it forces every subsequent case into the approval queue rather than auto-execution |
| FR-7.4 | Global kill switch | One control halts all autonomous action instantly; in-flight staged actions are cancelled; the event is logged with actor and timestamp |
| FR-7.5 | Reason-for-approval | The approval card shows the case, root cause with confidence, uplift, EV arithmetic, the exact policy rule requiring approval, the drafted message, and the projected cost — enough to decide in ten seconds |
| FR-7.6 | Idempotency guard | Every executable action carries a deterministic idempotency key `sha256(case_id ‖ action_type ‖ ladder_step ‖ policy_version)`; a duplicate key is a no-op that is still logged as a suppressed duplicate |
| FR-7.7 | Audit of overrides | Human approvals, rejections, cancellations and kill-switch events are first-class audited actions with actor identity |

---

### FR-8 Value and relationship-aware prioritization

> **Gap closed:** uniform treatment of every failed transaction is a functional defect, not a UX nicety. Chasing a five-year, high-LTV account the way you chase a chronic late payer destroys more value than it recovers.

| ID | Requirement | Detail |
|---|---|---|
| FR-8.1 | LTV and relationship score | Derived from historical order value, tenure, payment reliability, support history, and (for B2B) contract size and renewal proximity |
| FR-8.2 | Weighted aggressiveness | High-LTV accounts get gentler, earlier, more generous interventions and a lower contact cap; chronic late payers escalate faster **within** policy bounds — never beyond them |
| FR-8.3 | B2B ladder | Soft reminder → firm reminder with statement → *human-approved* formal notice → flagged for human collections. The last two steps are gated by FR-7.1 |
| FR-8.4 | Relationship-risk brake | If churn/relationship risk exceeds a threshold, escalation is capped and the case is routed to a human with a "handle personally" recommendation |
| FR-8.5 | Segment reporting | Recovery performance is broken down by segment so a merchant can see they are not winning ₹ from new customers while losing their best ones |

---

### FR-9 Multi-channel orchestration

> **Gap closed:** text-only, fixed-sequence dunning is the default build. Choosing *which* channel and *when*, per customer, under policy constraints, is where real recovery lift comes from.

| ID | Requirement | Detail |
|---|---|---|
| FR-9.1 | Channel-fit prediction | Predict the channel most likely to convert per customer from past engagement (delivery, open, click, reply, resolution) |
| FR-9.2 | **Constrained contextual bandit** | Channel + send-time selection uses Thompson sampling over the permitted action set. The bandit may only choose among actions the policy engine has already allowed and the EV gate has already cleared — exploration never escapes the guardrails |
| FR-9.3 | Channel fatigue | Track last-contacted channel and outcome; suppress a channel ignored twice consecutively; enforce per-channel cooldowns |
| FR-9.4 | Send-time optimization | Learned per-segment optimal hour within permitted hours (with a documented prior: salary-cycle timing for `insufficient_funds`) |
| FR-9.5 | Message generation | Claude drafts message copy from a case brief and a template contract; output is schema-validated, length-capped, PII-safe, tone-checked, and **never** sent without the policy engine's approval of the underlying action |
| FR-9.6 | Deterministic simulator | A full channel simulator with realistic, seeded response behaviour so the entire system is demonstrable and testable without spending a rupee or messaging a real person; adapters for live providers sit behind the same interface |
| FR-9.7 | Delivery-state tracking | Queued → staged → sent → delivered → engaged → converted / bounced / failed, each a logged case event |

---

### FR-10 Hinglish voice recovery

> **Gap closed:** voice is explicitly named in the brief, is materially harder than text, and is therefore where most teams will stop. Done narrowly and correctly, it is the most memorable thirty seconds of a pitch video.

| ID | Requirement | Detail |
|---|---|---|
| FR-10.1 | Bounded dialogue | A finite, auditable conversation graph — identify, state purpose, offer resolution, capture promise-to-pay or objection, confirm, close. No open-ended negotiation, no free-form commitments |
| FR-10.2 | Hinglish delivery | Natural code-mixed Hindi-English speech appropriate to Indian ops reality, with configurable language per customer |
| FR-10.3 | Disclosure and consent | The call opens by identifying the merchant and stating it is an automated assistant, and offers immediate opt-out and human transfer; both are honoured within the same call |
| FR-10.4 | Transcript as evidence | Full transcript, audio artifact, node path taken, and timestamps are attached to the case and hash-chained into the audit log |
| FR-10.5 | Safe degradation | On low ASR confidence, silence, hostility, or any out-of-graph turn, the agent apologizes, offers a link/callback, and ends the call — it never improvises |
| FR-10.6 | Escalation triggers | Distress, dispute, or legal keywords terminate the flow and raise a human exception |
| FR-10.7 | Cost-tier gating | Voice is the most expensive channel; the EV gate (FR-4) must clear it explicitly, so voice is reserved for high-value, high-uplift cases |

---

### FR-11 Promise-to-pay capture and trust scoring

> **Gap closed:** fire-and-forget reminders are the norm. Remembering what a customer committed to, and whether they honoured it, is cheap to build and almost nobody does it.

| ID | Requirement | Detail |
|---|---|---|
| FR-11.1 | PTP extraction | Extract a structured commitment `{amount, date, condition, confidence}` from voice transcripts and text replies using a constrained LLM extraction with a Pydantic schema; ambiguous extractions are marked low-confidence and human-verified |
| FR-11.2 | PTP honoured | An active PTP suspends escalation until the committed date + grace period; the case remains visible as `awaiting_promise`, not silently idle |
| FR-11.3 | Follow-through tracking | Compare committed date to actual resolution; log `kept`, `partial`, or `broken` |
| FR-11.4 | Trust score feedback | Kept promises increase patience and lower escalation aggressiveness for that customer; broken promises accelerate (within policy bounds) and reduce PTP-suspension grace next time |
| FR-11.5 | Extraction eval set | A golden set of ≥60 Hinglish/English utterances with labelled commitments; precision/recall reported, false-positive PTPs treated as the dangerous error class |

---

### FR-12 Self-serve recovery experience

> **Gap closed:** a reminder that ends in "please try again" makes the customer do the work. Recovery converts when the fix is one tap and the context is already loaded. This is also what turns the submission from a dashboard into a product a judge can *use*.

| ID | Requirement | Detail |
|---|---|---|
| FR-12.1 | Single-use recovery link | Every text-channel message contains a signed, single-use, expiring link to a hosted recovery page for that specific case |
| FR-12.2 | Context-aware page | The page states plainly what failed, why, what it costs, and offers the correct fix for that root cause: retry with the same method, switch method, update card, re-authorize mandate, or pay an invoice |
| FR-12.3 | Razorpay test-mode checkout | Resolution runs through Razorpay Payment Links / Checkout in test mode, so a judge can complete a payment on stage and watch the case flip to `recovered` live |
| FR-12.4 | One-tap preferences | The page offers opt-out and "remind me later" (setting a customer-chosen follow-up date that the policy engine honours) |
| FR-12.5 | Loop closure | Page views, method switches, opt-outs and completions flow back as case events and as bandit/uplift training signal |
| FR-12.6 | Accessibility and trust | Mobile-first, sub-second load, no dark patterns, merchant-branded, and explicit about what data is shown and why |

---

### FR-13 Incremental recovery measurement

> **Gap closed:** this is the module that most directly answers *"measured money recovered"*. A raw count of successful retries overstates impact because many of those payments would have succeeded anyway. This is the most defensible differentiator in the system — and the adaptive holdout is the answer to the obvious objection that a control group costs the merchant money.

| ID | Requirement | Detail |
|---|---|---|
| FR-13.1 | Randomized assignment at case creation | Eligible cases are randomly assigned `treatment` / `control` by a seeded, auditable hash of the case ID. The cohort is immutable and recorded before any scoring happens |
| FR-13.2 | Control integrity | A control case receives **zero** agent-initiated contact or retry. An invariant test proves no action can attach to a control case. Controls are still tracked to natural resolution |
| FR-13.3 | Stratified assignment | Randomization is stratified by root cause, amount band, and segment so the two arms are comparable at small batch sizes |
| FR-13.4 | Ethical / commercial guardrails on the holdout | Configurable holdout rate (default 20%); cases above a value threshold and cases at legal risk are excluded from control and always treated; exclusions are logged and reported |
| FR-13.5 | **Adaptive holdout** | The holdout rate decays automatically as evidence accumulates (sequential testing with an alpha-spending function); once the effect is established at the configured confidence, the holdout drops to a small monitoring slice. Honesty is a fixed cost, not a permanent tax |
| FR-13.6 | Incremental calculation | `lift = treated_resolution_rate − control_resolution_rate`; `incremental_₹ = lift × treated_volume × mean_recovered_value`, reported in ₹ and % over the batch window |
| FR-13.7 | Significance and interval | Two-proportion z-test with a 95% confidence interval on the lift, plus the minimum detectable effect for the batch size actually run; if the result is not significant, the dashboard says exactly that instead of rounding up |
| FR-13.8 | **Variance reduction** | CUPED-style adjustment using pre-period customer payment reliability as the covariate, to get a usable signal from a small batch; the unadjusted number is always shown alongside |
| FR-13.9 | Breakdowns | Lift by root cause, channel, segment, and value band — including any segment where lift is negative, reported without softening |
| FR-13.10 | Raw vs. incremental | Both numbers appear together, always, with the gap named on the dashboard: *"Raw recovered ₹X. Of that, ₹Y is attributable to Recoup."* |

---

### FR-14 Immutable audit trail and grounded explainability

> **Gap closed:** everyone will bolt an LLM chat onto a dashboard. That is not a differentiator. An explainability layer that is strictly grounded in a tamper-evident log and **refuses** to answer beyond it is.

| ID | Requirement | Detail |
|---|---|---|
| FR-14.1 | Event-sourced case store | Case state is derived from an append-only ordered event log; the current row is a projection, never the source of truth |
| FR-14.2 | Complete decision records | Every classification, score, EV computation, policy verdict (with rule ID and policy version), approval, action, delivery state and outcome is recorded with timestamp, actor, and input snapshot |
| FR-14.3 | Tamper-evident chain | Each event stores `prev_hash` and `hash = sha256(prev_hash ‖ canonical_payload)`; a verification command re-computes the chain and reports the first divergence |
| FR-14.4 | **Deterministic replay / time travel** | Any case can be replayed from its events to reconstruct exact state at any timestamp, and the whole batch can be re-derived from the log. Replay is a test, not just a feature |
| FR-14.5 | Grounded Q&A | An ops user asks "why did we call this account three times?" and receives an answer composed **only** from retrieved log entries, each claim carrying a citation to a specific event ID that the UI links to |
| FR-14.6 | Explicit refusal | If the answer is not in the log, the interface says so and names what would need to be logged to answer it. Speculation is a defect, not a fallback |
| FR-14.7 | Hallucination eval | A 40-question benchmark (20 answerable, 20 deliberately unanswerable) run in CI; fabricated citations fail the build |
| FR-14.8 | Export | One-click export of a case dossier or a whole batch (JSON + human-readable PDF/Markdown) for a compliance reviewer |

---

### FR-15 Merchant dashboard, simulator and reporting

> **Gap closed:** the dashboard is not the product, but it is the only part a judge experiences directly. It must make the differentiators visible in five seconds and let a sceptic interrogate them.

| ID | Requirement | Detail |
|---|---|---|
| FR-15.1 | Live batch summary | Revenue at risk, raw recovered, **incremental recovered with CI**, cost of recovery, cases by state, actions blocked by policy |
| FR-15.2 | Case work queue | Ranked by expected incremental value with a one-line human-readable reason and the SHAP/uplift evidence one click away |
| FR-15.3 | Approval and exception queues | Actions awaiting sign-off, cases at a stopping rule, low-confidence classifications, DLQ items, cases needing human handling |
| FR-15.4 | Case timeline | The full event-sourced story of one case, rendered as a vertical timeline with policy verdicts, costs, messages and outcomes inline |
| FR-15.5 | **What-if simulator** | Move the approval threshold, holdout rate, contact caps, EV floor or channel costs and see the projected effect on recovery, spend and contact volume, computed by replaying the historical log through the new policy |
| FR-15.6 | Compliance view | Live guardrail status: blocked actions with the rule that blocked them, opt-outs honoured, quiet-hours suppressions, mandate retries prevented |
| FR-15.7 | Model transparency | Confusion matrix, calibration curve, Qini curve, and a model card, in the product — not hidden in a notebook |
| FR-15.8 | Audit-ready export | One click produces the batch report used for submission |
| FR-15.9 | Live event stream | Server-sent events push case state changes to the UI so a demo shows the system working in real time |

---

### FR-16 Resilience, failure handling and chaos harness

> **Gap closed:** the brief explicitly values graceful failure and *"what broke and how you got out"*. Most submissions demo only the happy path. Recoup ships a button that breaks itself on stage.

| ID | Requirement | Detail |
|---|---|---|
| FR-16.1 | Chaos suite | Reproducible injection of: duplicate webhooks, out-of-order events, malformed payloads, provider 5xx and timeouts, LLM refusal/timeout/invalid-schema output, worker crash mid-action, clock skew, and a poisoned model output |
| FR-16.2 | Required outcomes | Every scenario must end with: zero duplicate customer contact, zero duplicate charge attempts, zero lost cases, and a truthful entry in the exception queue |
| FR-16.3 | Retry semantics | Exponential backoff with jitter and bounded attempts on every external call; failed external calls never mutate case state optimistically |
| FR-16.4 | LLM failure policy | If the model is unavailable, returns invalid schema, or exceeds latency budget, the system degrades to the deterministic path (template copy, rule-based routing) and marks the case `degraded_mode` — it never blocks recovery on model availability |
| FR-16.5 | Circuit breakers | Per-provider breakers with automatic half-open probing; an open breaker pauses that channel and reroutes rather than failing the case |
| FR-16.6 | Health and observability | `/health`, `/ready`, `/metrics`; structured JSON logs; OpenTelemetry traces spanning ingest → decide → act → measure |
| FR-16.7 | Demoable | A "Break it" control in the demo UI runs a chosen chaos scenario live and shows the system absorbing it |

---

## 7. Non-functional requirements

| Category | Requirement |
|---|---|
| **Financial safety** | No raw card PAN ever enters the system; tokenized references only; test-mode credentials only for v1; money-moving actions are idempotent, staged, and cancellable |
| **Correctness** | Policy engine is pure and deterministic; all money and time arithmetic uses exact types (`Decimal`, timezone-aware UTC internally, IST for display) |
| **Idempotency** | Every contact- or money-initiating action carries a deterministic idempotency key; duplicates are suppressed and logged, never executed |
| **Governance** | Merchant exposure cap, per-day spend cap, global kill switch, staged-action undo, full override audit |
| **Performance** | Ingest-to-decision p95 < 2s; grounded Q&A p95 < 4s; a 500-case batch processes end-to-end within the demo window |
| **Scalability** | Stateless API workers behind a queue; horizontal worker scaling; the demo bar (50+ records) is exceeded by an order of magnitude |
| **Reliability** | At-least-once delivery with exactly-once effects; every pipeline stage retryable; no case ever silently dropped |
| **Security** | Signed webhooks, secret management via env/secret store, least-privilege API keys, PII redaction before LLM calls, rate limiting, signed single-use recovery links |
| **Privacy** | Data minimization, configurable retention, customer-level export and deletion, no PII in logs or prompts |
| **Observability** | Structured logs, traces, metrics; every pipeline stage emits to the audit trail; **no side-channel unlogged actions** |
| **Testability** | Deterministic seeded synthetic data; a channel simulator; property-based invariant tests; replayable event log; CI gate on model metrics |
| **Maintainability** | Typed Python (mypy strict on core), ruff, ≥80% coverage on policy/economics/measurement modules, ADRs for every irreversible decision |
| **Accessibility** | Dashboard and recovery page meet WCAG AA contrast and keyboard navigation |

---

## 8. Data requirements

### 8.1 The `Case` entity (projection)

| Field | Description |
|---|---|
| `case_id` | ULID, stable, sortable by creation time |
| `merchant_id` | Tenant reference |
| `source_type` | `payment_failure` \| `checkout_abandonment` \| `mandate_failure` \| `receivable_overdue` |
| `provider_event_id` | Source event ID; unique per source (dedupe key) |
| `raw_event_ref` | Pointer to the stored raw payload |
| `amount_at_risk`, `currency` | Exact decimal value at risk |
| `customer_ref` | Tokenized customer/account reference — no PII beyond what test mode provides |
| `root_cause`, `root_cause_confidence`, `root_cause_features` | FR-2 output including SHAP attribution |
| `p_recover_baseline`, `p_recover_treated`, `uplift`, `uplift_segment` | FR-3 output |
| `ltv_score`, `relationship_risk`, `trust_score` | FR-8 / FR-11 output |
| `cohort` | `treatment` \| `control` — set at creation, immutable, with assignment seed recorded |
| `escalation_state`, `policy_version` | Current ladder position and the policy that governs it |
| `contact_budget_used` | Contacts consumed against the rolling per-customer budget |
| `ev_ledger[]` | Every EV computation with inputs and verdict |
| `action_log[]` | Ordered actions: type, channel, idempotency key, cost, staged/sent/cancelled, outcome |
| `ptp` | Promise-to-pay: amount, date, condition, confidence, follow-through outcome |
| `resolution_state` | `recovered` \| `pending` \| `awaiting_promise` \| `stopped_by_policy` \| `abandoned_uneconomic` \| `exception` \| `control_untouched` |
| `resolved_at`, `recovered_amount` | Outcome facts |

### 8.2 The `CaseEvent` entity (source of truth)

`event_id`, `case_id`, `seq`, `occurred_at`, `recorded_at`, `actor` (`system` \| `model` \| `human:<id>` \| `provider`), `event_type`, `payload` (canonical JSON), `policy_version`, `model_versions`, `prev_hash`, `hash`.

### 8.3 Supporting entities

`Customer` (tokenized ref, consent per channel, DND status, opt-out timestamp, contact history, trust score), `Merchant` (config: thresholds, caps, costs, margin, kill switch), `PolicyVersion` (YAML content hash, adopted-at, adopted-by), `ModelVersion` (artifact hash, training data hash, metrics, model card), `ChannelMessage` (template ID, rendered body hash, delivery states), `Experiment` (batch window, holdout rate schedule, results).

### 8.4 Synthetic data requirements

- ≥ 2,000 labelled training cases and a ≥ 500-case demo batch (the stated bar is 50+; exceeding it by 10× is deliberate).
- Realistic Indian failure-mix distributions, issuer/bank variety, amount distributions, salary-cycle effects, and a **ground-truth response curve per case** so uplift models and the simulator can be validated against known truth.
- Fully seeded and reproducible: `make data SEED=42` regenerates the identical batch. The generator, distributions, and their justifications are documented in the repo.

---

## 9. Primary workflow

1. A revenue-at-risk event arrives (FR-1), is signature-verified, deduplicated, and normalized into a `Case`.
2. The case is assigned to `treatment` or `control` by stratified randomization, immutably, before any scoring (FR-13.1).
3. **If control:** no agent action, ever. The case is tracked passively to natural resolution for the baseline.
4. **If treatment:** root cause is classified with calibrated confidence and SHAP attribution (FR-2). Low confidence → conservative ladder + human flag.
5. Baseline propensity, treated propensity and uplift are computed; the case is bucketed into a targeting segment (FR-3). Sure things, lost causes and sleeping dogs are marked `no_action_recommended`.
6. LTV, relationship risk and trust weight the priority score (FR-8, FR-11).
7. The policy engine enumerates the permitted next actions for this cause, state, consent, hour and mandate status, returning ALLOW / DENY / REQUIRE_APPROVAL with rule IDs (FR-5, FR-6).
8. The economic layer computes EV for each permitted action and drops everything below the floor; if nothing clears, the case terminates as `abandoned_uneconomic` with the arithmetic logged (FR-4).
9. The bandit selects channel and send-time among the surviving actions (FR-9.2).
10. If the action requires approval or exceeds the exposure cap, it enters the approval queue with a decision card (FR-7.1, FR-7.5).
11. The action is **staged** with an idempotency key and an undo window, then executed through the chosen channel — text with a single-use recovery link (FR-12), or the bounded Hinglish voice flow (FR-10).
12. Inbound signals — delivery, open, click, page view, reply, PTP, partial payment, opt-out — flow back as case events, re-scoring the case (FR-3.6) and updating the bandit and trust score.
13. Every step above appends to the hash-chained event log (FR-14).
14. The case terminates in exactly one resolution state; nothing is silently dropped (FR-16).
15. At batch end the measurement engine compares treatment against control, applies CUPED adjustment, runs the significance test, and produces the headline incremental figure with its confidence interval and breakdowns (FR-13).
16. Results, exceptions, compliance status, model transparency and the audit trail surface on the dashboard, exportable in one click (FR-15).

---

## 10. Compliance and governance requirements

- **C1** No autonomous action may exceed the merchant-configured exposure cap without human approval.
- **C2** No contact outside permitted hours, to an opted-out customer, or on a non-consented channel.
- **C3** Recurring-mandate retries follow the configured cadence and never silently retry a case requiring customer re-authorization.
- **C4** B2B escalation beyond a firm reminder requires human sign-off before execution; Recoup drafts formal notices but never sends them autonomously.
- **C5** No raw card data; tokenized references only; PII redacted before any model call.
- **C6** Every action is attributable: actor, timestamp, policy version, model version, idempotency key.
- **C7** Recoup is **recovery-only and strictly defensive**. It never applies pressure, deception, urgency manipulation, false consequence, or any dark pattern. Message copy is checked against a prohibited-claims list before dispatch. If an action could plausibly be repurposed to harass, it is not implemented.
- **C8** Every regulatory constant is named, cited, owned, test-covered, and flagged for re-verification before production use. Recoup claims *alignment by design*, not legal certification.

---

## 11. Trust and safety: the agent authority model

| Capability | LLM may | Deterministic code must |
|---|---|---|
| Classify root cause | Assist / explain | Own the decision (trained model) |
| Score uplift and EV | Never | Own entirely |
| Decide whether an action is permitted | Never | Own entirely (policy engine) |
| Choose channel and timing | Never | Own (bandit under policy constraints) |
| Draft message copy | Yes, schema-validated and safety-checked | Approve, render from template, and dispatch |
| Conduct a voice call | Yes, inside a bounded graph | Own the graph, the exits, and the recording |
| Extract a promise-to-pay | Yes, into a strict schema | Validate, threshold, and gate low confidence to a human |
| Answer "why did this happen?" | Yes, from retrieved log entries only | Provide retrieval, enforce citations, enforce refusal |
| Execute a retry, a charge, or a contact | **Never** | Own entirely |

Failure of the model layer degrades the product's polish. It can never degrade its safety.

---

## 12. Assumptions and constraints

- All payment, mandate and receivables data for the build is synthetic or Razorpay test-mode; no real PII, no live money movement.
- The demo batch is ≥ 500 cases (bar: 50+).
- Live SMS/WhatsApp/voice run on free-tier/trial credentials where available; the deterministic simulator is the default path so the system is fully demonstrable offline and at zero cost. Both sit behind one interface.
- The Hinglish voice flow is a narrow, pre-defined conversation graph, not open-ended negotiation, to keep it auditable within the build window.
- Uplift models are bootstrapped from the synthetic generator's ground-truth response curves and are honestly labelled as such; the production path (learning uplift from accumulated live holdout data) is documented in the architecture.
- Regulatory parameters are conservative defaults requiring re-verification against current circulars before production use.
- Build window: 8 days, solo builder, backend-weighted with a high-quality UI.

---

## 13. Out of scope for v1

- Fraud detection, disputes, chargebacks, or any offence-capable analysis (Track 02).
- Legal or formal collections action beyond generating a human-approval request and a draft.
- Production bank-rail integration or live money movement.
- Free-form conversational negotiation of payment terms.
- Multi-language beyond Hinglish/English for voice, and beyond English for text templates.
- Full multi-tenant billing, SSO, and RBAC beyond a single-merchant demo tenant with an ops role.

---

## 14. Acceptance criteria (definition of done)

| # | Criterion | Evidence required |
|---|---|---|
| 1 | Four event sources ingest into one normalized case | Live demo of all four; DLQ shown non-empty and handled |
| 2 | Root-cause classifier evaluated on a held-out test set | Per-class precision/recall/F1, confusion matrix, calibration curve, Brier score, and an honest list of failure modes |
| 3 | Uplift targeting demonstrably avoids waste | Qini curve; a case visibly skipped as `sure_thing` and one as `sleeping_dog`, with ₹ saved reported |
| 4 | EV gate demonstrably refuses an uneconomic chase | A case terminating `abandoned_uneconomic` with its arithmetic on screen |
| 5 | Incremental recovery reported with a confidence interval | Treatment vs. control breakdown, z-test, CI, MDE, CUPED-adjusted and unadjusted, in ₹ and % |
| 6 | Every executed action is idempotent and logged | Replay a duplicate webhook live; show zero duplicate actions and a logged suppression |
| 7 | Stopping rules and regulatory guardrails enforced | ≥3 blocked actions demonstrated: quiet hours, opt-out, revoked-mandate retry — each naming the rule ID |
| 8 | Human authority is real | Approval card approved and a staged action cancelled inside its undo window; kill switch halts everything live |
| 9 | One failure handled gracefully end-to-end | Chaos scenario run on stage; case lands in the exception queue with a truthful reason and no duplicate side effects |
| 10 | Audit trail fully reconstructs any case | Hash-chain verification passes; deterministic replay reproduces state; grounded Q&A answers with citations **and refuses** an out-of-log question |
| 11 | Hinglish voice flow demonstrated | Recorded call with transcript, node path, and a captured promise-to-pay linked to the case |
| 12 | Self-serve recovery works | Judge opens the link and completes a test-mode payment; the case flips to `recovered` live on the dashboard |
| 13 | Reproducibility | `make demo` regenerates the identical batch and the identical headline numbers from seed 42 on a clean machine |
| 14 | Honest engineering narrative | `09-INCIDENT-LOG.md` documents what broke during the build and how it was diagnosed and fixed |

---

## 15. Key risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Small batch → statistically insignificant lift | Headline number is weak | Large synthetic batch, stratified assignment, CUPED, and — critically — **report the insignificance honestly** with the MDE. A defensible null beats an indefensible win |
| Synthetic data flatters the models | Judges discount results | Publish the generator, its distributions, and its ground-truth curves; report metrics on held-out data; never claim real-world performance |
| Scope overrun in 8 days | Nothing finishes well | Phase plan with hard cut lines; the demo-critical path (FR-1, 2, 5, 6, 7, 13, 14) is protected; voice, bandit and simulator are the first things descoped |
| Live provider integrations break on demo day | Demo failure | Simulator is the default path; live providers are an optional flag, never a dependency |
| LLM latency or refusal mid-demo | Awkward pause | Deterministic degradation path (FR-16.4) with pre-warmed cache for demo cases |
| Regulatory claims overstated | Credibility loss with a fintech judge | Every constant is configurable, cited, and explicitly flagged as requiring verification; claim alignment, never certification |

---

## 16. Glossary

| Term | Meaning |
|---|---|
| **Case** | Recoup's normalized unit of work: one revenue-at-risk event |
| **Uplift (τ)** | P(recover \| treated) − P(recover \| untreated); the only recovery number worth optimizing |
| **Persuadable / sure thing / lost cause / sleeping dog** | The four uplift segments; only persuadables justify contact |
| **Incremental recovery** | Recovered revenue attributable to the agent, beyond what would have recovered naturally |
| **Treatment / control cohort** | The randomized split that makes recovery measurable causally rather than descriptively |
| **Adaptive holdout** | A control group whose size decays as statistical evidence accumulates |
| **CUPED** | Controlled-experiment variance reduction using pre-period covariates; more signal from fewer cases |
| **EV gate** | The rule that no action is taken whose expected value is below the configured floor |
| **Escalation ladder** | The ordered set of permitted interventions for a failure type, bounded by stopping rules |
| **Staged action** | An approved action held in a cancellable state for a configured window before execution |
| **PTP** | Promise-to-Pay: a customer-stated commitment to resolve by a date or condition |
| **Policy-as-code** | Escalation and guardrail rules declared in versioned, tested YAML rather than scattered in application code |
| **Idempotency key** | Deterministic identifier guaranteeing an action can never execute twice |
| **Hash chain** | Linked hashes over the event log making silent tampering detectable |
| **Grounded Q&A** | Natural-language answers composed only from retrieved log entries, with citations and mandatory refusal outside them |
