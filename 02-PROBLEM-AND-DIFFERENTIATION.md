# Recoup — Problem, User Impact and Differentiation

**Why this problem is real, why merchants badly want it solved, and why an AI wrapper won't solve it**

| Field | Detail |
|---|---|
| Document | Companion to `01-FRD.md` |
| Version | **2.0** (supersedes v1.0, August 2026) |
| Track | Razorpay AI Buildathon — Track 03: AI Revenue Recovery |
| Audience | Judges, reviewers, and anyone deciding in five minutes whether this is a product or a demo |

---

## 1. Why this document exists

The FRD specifies *what* Recoup must do. This document answers the question that comes first: **does anyone actually need it, and would a team that simply asks an AI agent to "build a payment recovery bot" end up with the same thing?**

It walks through the problem twice — once the way a merchant would describe it, once the way an engineer would — quantifies who is hurt and by how much, and then makes the differentiation case explicitly, feature by feature, against the submission we expect most teams to produce.

---

## 2. The problem, in plain language

### 2.1 A story, not a spec

Priya runs a mid-sized online store. Every month, of every ₹100 customers *try* to pay her, roughly ₹8–₹15 never reaches her account — not because customers don't want to pay, but because something got in the way. The bank's OTP page timed out. The card had expired. The UPI Autopay renewal for a subscription silently stopped working. The customer got distracted at checkout and never came back. Separately, a handful of B2B buyers simply haven't paid their invoice yet, and nobody on her four-person team has time to chase each one.

Today Priya has three bad options:

1. **Ignore it.** Accept the leak as a cost of doing business. Most merchants do this, because the alternatives are worse.
2. **Chase it manually.** Hire someone to call and message everyone. This barely scales past a few dozen cases a day, and it treats a loyal five-year customer exactly like a stranger.
3. **Install a generic reminder tool.** It blasts the same SMS to everyone on a timer, annoys good customers, ignores India's do-not-disturb rules, and — the part that really matters — cannot tell her whether it made any difference or whether those customers would have paid anyway.

> **The part that's easy to miss**
>
> The hardest part of this problem is not sending a reminder. It is knowing **which customer to leave alone**, which one to nudge gently, which one needs a completely different fix (a mandate re-authorization link, not a reminder), which one isn't worth the ₹3 it costs to chase — and then being able to prove, in real numbers, that the effort actually made money rather than just felt productive.

### 2.2 What Priya would say she wants, in her words

These are the things merchants ask for out loud, and each maps to a specific requirement rather than a slogan:

| What she says | What it means in the build |
|---|---|
| *"Don't annoy my good customers."* | Uplift segmentation + relationship weighting + contact-fatigue budget (FR-3, FR-4.5, FR-8) |
| *"Don't do anything expensive without asking me."* | Approval gate, exposure cap, staged actions with an undo window, kill switch (FR-7) |
| *"Tell me it actually worked — and don't lie to me."* | Incremental recovery vs. control, with a confidence interval and the raw number shown beside it (FR-13) |
| *"I don't want a legal problem."* | Regulatory guardrails separated from business config, each mapped to a test (FR-6, `06-COMPLIANCE-MATRIX.md`) |
| *"Why did it call this customer three times?"* | Grounded Q&A over a tamper-evident log, with citations and refusal (FR-14) |
| *"Just let them fix it in one tap."* | Single-use, context-aware self-serve recovery page with test-mode checkout (FR-12) |
| *"What if I loosened this rule?"* | What-if simulator replaying history through a new policy (FR-15.5) |
| *"What happens when it breaks at 2am?"* | Chaos harness, DLQ, exception queue, degraded mode, no silent drops (FR-16) |

Every one of these is a feature a real user asks for unprompted, and almost none of them appear in a bot built from a generic prompt.

---

## 3. The same problem, in technical language

Revenue leakage across a payments stack decomposes into four structurally different failure classes, each with a different optimal remediation and a different regulatory envelope:

| Failure class | Root cause examples | Correct remediation |
|---|---|---|
| **One-time payment failure** | Bank soft decline, OTP timeout / auth abandon, insufficient funds, issuer risk block, gateway/network error, expired card | Root-cause-specific retry timing or channel nudge — never a blanket retry |
| **Checkout abandonment** | Customer drop-off pre-payment; no failure event exists at all | Timed, low-friction re-engagement with a direct resume link |
| **UPI Autopay / e-mandate failure** | Technical failure (retryable) vs. revoked mandate or insufficient balance (needs re-authorization or notification) | Cadence-compliant retry, or a re-authorization flow — never a silent blind retry |
| **B2B receivable overdue** | Invoice simply unpaid past due date | Relationship-aware escalation ladder with a human-approval gate before anything formal |

A system that treats all four as *"payment failed → send reminder"* is solving the wrong-shaped problem. Built without deliberate constraints it will also violate real rules — mandate retry cadence, DND and consent, data protection — and break basic engineering invariants: a duplicate webhook double-charging a customer, an out-of-order success event triggering a reminder for a payment that already went through. None of these are edge cases in production. **They are the default failure modes of a naive implementation.**

There is a second technical error, subtler and more expensive: **optimizing the wrong quantity.** A retry bot that maximizes "recovered payments" will happily spend its entire budget on the cases that were going to recover on their own, because those are the easiest to "recover". The correct objective is not P(recover) but **uplift** — P(recover | we act) − P(recover | we don't) — and the correct measurement is the same quantity at the batch level. Almost nobody builds it this way, because you only notice the error if you have already thought about the control group.

---

## 4. Why "just ask an AI to build it" doesn't close the gap

By the time this buildathon is judged, essentially every team will have used an AI coding agent to scaffold their submission. That is the baseline, not a differentiator. Ask any current model to *"build an AI agent that recovers failed payments"* and it will competently produce a webhook listener, a retry scheduler, an LLM-written reminder, and a dashboard showing "₹X recovered." That is a real, working demo — and it is exactly what a large fraction of the field will have, because it is the first thing any model reaches for when the prompt is generic.

The gap is not code quality. **The gap is in the requirements nobody thinks to ask for**, because they are not obvious unless you already know the domain:

1. An AI agent will not spontaneously **hold out a control group** to measure whether its actions caused the recovery. It will report the raw number, because that is what "show money recovered" naively means.
2. An AI agent will not spontaneously distinguish **uplift from propensity**, so it will proudly chase the customers who were going to pay anyway.
3. An AI agent will not spontaneously **price its own actions**, so it will spend ₹4 chasing ₹60 and call it a win.
4. An AI agent will not spontaneously know that **recurring-mandate failures have a regulator-defined cadence** distinct from a card decline. It will retry both the same way unless told not to.
5. An AI agent will not spontaneously add a **human-approval gate and a cancellable window** before an action crosses into something that looks like collections. It will automate the whole ladder end-to-end.
6. An AI agent will not spontaneously add an **idempotency key**, so a duplicate webhook becomes a duplicate charge attempt. It builds the happy path first, always.
7. An AI agent's chat layer will, by default, **generate a plausible justification for any question**, including questions the system never logged an answer to — unless it is explicitly constrained to cite real log entries and refuse otherwise.
8. An AI agent will not spontaneously build a **chaos harness that attacks its own happy path**, because nothing in the prompt asked it to try to break itself.

> **The standout strategy in one sentence**
>
> We are not trying to out-code the other teams' AI agents — every agent writes good code. We are handing our agent a specification that already contains the eight things a generic prompt would never think to ask for, so the resulting product is *structurally* different, not just prettier.

---

## 5. What we're building — explained twice

Every feature below is explained once for a non-technical reader and once for an engineer, followed by the specific gap it closes.

### 5.1 Root-cause diagnosis (not just "payment failed")

| Lens | Explanation |
|---|---|
| **In plain terms** | Instead of treating every failed payment the same, Recoup first works out *why* it failed — a bad OTP, an expired card, no money in the account, a cancelled subscription mandate — because the right next step is completely different in each case. |
| **Technically** | A calibrated gradient-boosted multi-class classifier labels each event into a decline taxonomy using transaction metadata, decline codes and customer history; evaluated on a held-out test set with per-class precision/recall, a reliability curve and a Brier score; SHAP attribution attached to every case. |
| **Gap it closes** | Generic bots bucket every failure into one "retry + remind" path. This is the foundation that makes every downstream decision correct instead of generic. Calibration matters because the economics layer consumes these probabilities as real numbers. |

### 5.2 Uplift-based targeting (the model that decides who to leave alone)

| Lens | Explanation |
|---|---|
| **In plain terms** | Some customers will pay on their own — chasing them wastes money and goodwill and then takes credit for their payment. Some will never pay whatever you do. A few are genuinely persuadable. Recoup predicts the *difference* our action makes, and spends effort only where that difference is positive. |
| **Technically** | Two-model / X-learner uplift estimation of τ = P(recover \| treated) − P(recover \| untreated), bucketing cases into sure things, persuadables, lost causes and sleeping dogs; evaluated with a Qini curve and uplift-at-k on held-out data. |
| **Gap it closes** | This is the model-level twin of the control group and the single most sophisticated idea in the build. It converts "we recovered a lot" into "we caused a lot", at the level of the individual decision rather than only the batch report. |

### 5.3 Economics: knowing what an action costs

| Lens | Explanation |
|---|---|
| **In plain terms** | Every message, WhatsApp template and phone call costs real money — and every extra contact costs goodwill. Recoup does the arithmetic before it acts: if chasing ₹60 will cost ₹4 and only has an 8% chance of making a difference, it doesn't chase, and it shows the merchant the sum it just did. |
| **Technically** | `EV = uplift × amount × margin − channel_cost − goodwill_cost(contact_n)`, with an EV floor, a per-customer rolling contact-fatigue budget, and merchant-level daily spend caps; cases with no positive-EV action terminate as `abandoned_uneconomic` with the ledger recorded. |
| **Gap it closes** | Almost no recovery bot — commercial ones included, at the SME tier — knows the price of its own behaviour. "Cost per ₹ recovered" is the metric a finance lead actually buys on, and it is the metric that makes restraint look like intelligence rather than laziness. |

### 5.4 Compliant, bounded escalation (the part most bots skip)

| Lens | Explanation |
|---|---|
| **In plain terms** | Recoup never contacts or retries without limits. Don't call outside allowed hours. Don't contact someone who opted out. Don't retry a subscription payment in a way that breaks the rules. Always stop after a fixed number of tries. These are rules the merchant can read, not behaviour buried in code. |
| **Technically** | A pure, deterministic policy engine evaluates versioned YAML policy-as-code and returns ALLOW / DENY / REQUIRE_APPROVAL with the exact rule ID; regulatory guardrails (quiet hours, consent, DND, mandate cadence, re-authorization, data protection) are kept in a separate, non-merchant-tunable layer; property-based tests assert the invariants over randomly generated histories. |
| **Gap it closes** | This is the most commonly skipped requirement in a generic build and precisely what *"compliant escalation, stopping rules"* in the brief is testing for. Policy-as-code with invariant tests is also the clearest possible signal of fintech engineering maturity. |

### 5.5 Human authority and real reversibility

| Lens | Explanation |
|---|---|
| **In plain terms** | Anything expensive or serious waits for a human to press approve. Even approved actions sit in a short cancellable window before they go out, so a mistake can be pulled back. One switch stops everything instantly. |
| **Technically** | Approval gate above a configurable value/risk threshold, staged-action buffer with a configurable undo window, merchant exposure cap that forces queueing when exceeded, a global kill switch that cancels in-flight staged actions, and a decision card that shows the case, cause, uplift, EV arithmetic, drafted copy and triggering rule. |
| **Gap it closes** | Every agent project *claims* actions are reversible. Almost none specify the mechanism. "Reversible before it becomes irreversible" is either a state machine or a slogan; here it is a state machine. |

### 5.6 Value and relationship-aware prioritization

| Lens | Explanation |
|---|---|
| **In plain terms** | A five-year loyal customer who missed one payment should get a gentle nudge, not the same treatment as someone who has ignored three invoices. Recoup treats people differently based on their relationship with the business, the way a good ops person would. |
| **Technically** | An LTV / relationship-risk score weights escalation aggressiveness and contact caps; B2B receivables follow a soft-reminder → firm-reminder → human-approved-formal-notice ladder; a relationship-risk brake caps escalation and routes to a human when churn risk is high. |
| **Gap it closes** | Most recovery bots are transaction-uniform. This is the difference between a script and a system that reasons about the business relationship rather than the transaction. |

### 5.7 Multi-channel orchestration and Hinglish voice recovery

| Lens | Explanation |
|---|---|
| **In plain terms** | Recoup picks the channel a specific customer is actually likely to respond to — SMS, WhatsApp, email, or a real Hinglish phone call — instead of blasting every channel, and it backs off from a channel that keeps getting ignored. |
| **Technically** | A channel-fit model plus a **constrained contextual bandit** (Thompson sampling) over channel and send-time, which may only explore among actions the policy engine has already permitted and the EV gate has already cleared; channel-fatigue suppression; and a bounded, auditable Hinglish conversation graph for reminder and re-authorization calls with disclosure, opt-out, human transfer and safe degradation. |
| **Gap it closes** | Text-only reminder bots are the default. A working voice flow is explicitly named in the brief and is materially harder to build well — which is exactly why few teams will attempt it, and why it stands out when done narrowly and correctly. The bandit-inside-guardrails design is also the honest answer to "is your agent actually learning?": yes, and it still cannot escape the rules. |

### 5.8 Promise-to-pay memory

| Lens | Explanation |
|---|---|
| **In plain terms** | If a customer says *"I'll pay by Friday"*, Recoup remembers, goes quiet until Friday, checks whether the promise was kept, and adjusts how it treats that customer next time — patient with people who keep their word, faster with people who don't. |
| **Technically** | Commitments are extracted from voice and text into a strict schema with a confidence threshold, suspend escalation until the committed date plus grace, are compared against actual resolution, and feed a trust score that modulates FR-8 aggressiveness. Evaluated against a labelled golden set, with false-positive PTPs treated as the dangerous error class. |
| **Gap it closes** | Fire-and-forget reminders are the norm. Closing this loop is cheap to build, high-value, and almost nobody includes it — and "the system went quiet because the customer promised Friday" is a moment that lands instantly with any judge who has ever done collections. |

### 5.9 Self-serve recovery in one tap

| Lens | Explanation |
|---|---|
| **In plain terms** | Every message ends in a link that opens a page which already knows what went wrong and offers exactly the right fix — retry, switch method, update card, re-authorize the subscription, or pay the invoice — in one tap. |
| **Technically** | Signed, single-use, expiring links to a context-aware recovery page backed by Razorpay test-mode Payment Links / Checkout; page events flow back as case events and as learning signal; opt-out and "remind me later" are first-class and honoured by the policy engine. |
| **Gap it closes** | Reminders that end in "please try again" make the customer do the work. This is also what converts the submission from a dashboard into a product a judge can *use*: they open the link on their own phone, pay in test mode, and watch the case flip to recovered live. |

### 5.10 Proof that it actually worked

| Lens | Explanation |
|---|---|
| **In plain terms** | Instead of counting how many reminded customers eventually paid — some of whom would have paid anyway — Recoup deliberately leaves a small, comparable group untouched and compares. The difference is the honest amount of money the system *caused* to come back. And once that difference is proven, the untouched group shrinks automatically, so honesty stops costing the merchant money. |
| **Technically** | Stratified randomized treatment/control assignment at case creation, immutable and enforced by an invariant test; incremental recovery = treated rate − control rate reported in ₹ and % with a two-proportion z-test, 95% CI, and the minimum detectable effect for the batch actually run; CUPED variance reduction using pre-period reliability; **adaptive holdout** with alpha-spending that decays the control share as evidence accumulates; breakdowns by cause, channel, segment and value band, including negative ones. |
| **Gap it closes** | This is the literal answer to *"show measured money recovered"*. Most teams will report a raw, uncontrolled number. This is the one figure a judge cannot poke a hole in — and the adaptive holdout pre-empts the obvious counter-question ("isn't your control group costing the merchant money?") before it is asked. |

### 5.11 An explainability layer that refuses to guess

| Lens | Explanation |
|---|---|
| **In plain terms** | You can ask Recoup, in plain English, *"why did we contact this customer three times?"* and it answers from its own records, showing you the records. If the answer isn't in its records, it says so instead of inventing something. |
| **Technically** | Every decision and action is an event in an append-only, hash-chained log from which case state is deterministically replayed; grounded Q&A retrieves and cites specific event IDs, and a CI-enforced 40-question benchmark (20 answerable, 20 unanswerable) fails the build on a fabricated citation or a missing refusal. |
| **Gap it closes** | An LLM chat window bolted onto a dashboard is not a differentiator — everyone will have one. A chat layer that is provably grounded, cites primary records, and refuses to speculate is the part that is hard to get right and the only version that would survive a compliance review. |

### 5.12 Breaking it on purpose

| Lens | Explanation |
|---|---|
| **In plain terms** | There is a button in the demo that deliberately breaks the system — duplicate events, a provider outage, a crash mid-action — so you can watch it not double-charge anyone, not lose anything, and put the problem where a human will see it. |
| **Technically** | A reproducible chaos suite (duplicate/out-of-order/malformed events, provider 5xx, LLM timeout and invalid schema, worker crash mid-action, clock skew) asserted to produce zero duplicate contacts, zero duplicate charge attempts, zero lost cases, and a truthful exception entry; plus circuit breakers, bounded backoff, a dead-letter queue, and a deterministic degraded mode when the model is unavailable. |
| **Gap it closes** | The brief explicitly rewards graceful failure and asks *"what broke and how you got out"*. Most submissions demo only the happy path and answer that question with an anecdote. Ours answers it with a test suite and an incident log. |

---

## 6. Recoup vs. a generic AI-scaffolded submission

| Dimension | Generic AI-scaffolded recovery bot | Recoup |
|---|---|---|
| Failure handling | One bucket: "payment failed" → retry + remind | Four failure classes, each with its own remediation and regulatory envelope |
| Objective function | Maximize successful retries | Maximize **incremental** recovered value net of cost |
| Targeting | Chase everything, hardest cases first | Uplift segmentation; sure things, lost causes and sleeping dogs deliberately left alone |
| Cost awareness | None | Per-action cost model, EV gate, goodwill cost, cost-per-₹-recovered as a headline metric |
| Reported impact | Raw count of eventually-successful retries | Incremental lift vs. a stratified control, with z-test, CI, MDE, CUPED — and the raw number shown beside it |
| Control group | None | Adaptive holdout that shrinks as evidence accumulates |
| Contact limits | Timer-based retries, no hard ceiling | Hard stopping rules, per-customer rolling fatigue budget, quiet hours, consent and opt-out enforced at the policy layer |
| Regulatory posture | Not considered | Regulatory guardrails separated from business config, each mapped to a named constant and a test |
| Risk control | Fully automated end-to-end | Approval gate, staged actions with an undo window, exposure cap, kill switch, idempotency guard |
| Personalization | Same treatment for everyone | LTV/relationship weighting, trust score, promise-to-pay memory |
| Channels | Text-only (SMS/email) | SMS, WhatsApp, email, self-serve page, bounded Hinglish voice — chosen by a bandit inside the guardrails |
| Customer experience | "Payment failed, please try again" | One-tap, context-aware recovery page with the correct fix pre-selected |
| Explainability | LLM chat that can improvise | Grounded Q&A citing a hash-chained log, with enforced refusal and a CI benchmark |
| Failure handling (engineering) | Happy path only | Chaos suite, DLQ, circuit breakers, degraded mode, exception queue, deterministic replay |
| Reproducibility | "It worked on my laptop" | `make demo` reproduces the identical batch and headline numbers from a seed |

---

## 7. What the judges are actually testing, line by line

| What the brief says | The trap | What Recoup shows |
|---|---|---|
| *"detects revenue at risk"* | Only handling `payment.failed` | Four sources, one normalized case, DLQ for the rest |
| *"determines the right intervention"* | One ladder for everything | Cause-specific ladders + uplift + EV + bandit under constraints |
| *"executes a bounded recovery workflow"* | Unbounded automation | Hard stopping rules, approval gate, staged undo, exposure cap, kill switch |
| *"show measured money recovered"* | A big raw number | Incremental ₹ with CI, z-test, MDE, and the raw number shown for contrast |
| *"across a batch"* | 20 cherry-picked cases | 500-case seeded batch, reproducible from scratch |
| *"compliant escalation"* | Compliance mentioned in the README | Regulatory constants, named, cited, test-covered, in a compliance matrix |
| *"stopping rules"* | `if attempts > 3: stop` | Policy-as-code with property-based invariant tests and a policy simulator |
| *"audit trail"* | A logs table | Event-sourced, hash-chained, deterministically replayable, grounded Q&A with refusal |
| *"honest metrics" / "graceful failure"* | Happy-path demo | Chaos suite on stage, exception queue, degraded mode, published failure modes, incident log |

---

## 8. Why this is worth an internship, not just a prize

The hiring question behind this event is not *"can you ship in a weekend?"* — every shortlisted candidate can. It is *"would I put this person near money in production?"* Recoup is engineered to answer that specific question:

- **It says no.** The most important behaviour in the system is refusal: refusing to contact, refusing to retry, refusing to spend, refusing to answer beyond the log. Systems that only know how to act are the ones that cause incidents.
- **It measures itself honestly, including when the answer is unflattering.** Reporting a non-significant lift with its confidence interval is a stronger signal of judgment than a large number with no error bar.
- **It separates what an LLM may do from what code must do**, and writes that boundary down. That single table is the difference between an agent you can deploy and a demo you can't.
- **It is reproducible.** A seed, a Makefile, and a clean machine produce the same numbers. Anyone can verify the claims without trusting the author.
- **It documents its own failures.** The incident log is not an apology; it is the artifact that shows how the builder debugs under pressure — which is the thing the brief explicitly asks about.

---

## 9. Conclusion

The problem Recoup addresses is not hypothetical. It is the same leak every digital-first merchant in India already lives with, and it is large enough that entire funded companies exist to plug pieces of it. The reason it remains unsolved for small and mid-sized merchants is not a lack of AI. It is that solving it well requires knowing which unglamorous requirements to insist on *before* you start prompting a coding agent.

This document exists to make sure those requirements — **causal measurement, uplift-based targeting, cost-aware restraint, compliant stopping rules, real reversibility, closed-loop promise tracking, grounded explainability, and deliberate self-inflicted failure** — are decided on purpose, rather than discovered missing the night before the demo.
