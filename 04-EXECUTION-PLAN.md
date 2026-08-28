# Recoup — Execution Plan for Claude Code

**The master build document. Hand this, plus `context/phase-NN-*.md`, to Claude Code and build in order.**

| Field | Detail |
|---|---|
| Version | 2.0 |
| Window | 8 days (build starts Day 1; submission lands Day 8, ahead of the 5 September deadline) |
| Team | Solo builder, backend-weighted, with a deliberately high-quality UI |
| Companion docs | `01-FRD.md` (what), `03-ARCHITECTURE.md` (how it fits), this (in what order), `05-EVALUATION-PROTOCOL.md` (how we prove it) |

---

## 1. How to use this document

This plan assumes **one Claude Code session per phase**, started clean. Long sessions drift; fresh sessions with tight context do not.

The loop for every phase:

```bash
# 1. Start a fresh Claude Code session in the repo root
claude

# 2. Prime it — CLAUDE.md loads automatically; add the phase context explicitly
> Read context/shared/*.md and context/phase-04-policy-engine.md, then restate
> the phase objective, the definition of done, and your implementation plan
> before writing any code.

# 3. Approve the plan, then build
# 4. Run the phase gate
make gate PHASE=04

# 5. Commit with the phase trailer, then close the session
```

**Rules that make this work:**

- Never start a phase whose predecessor's gate is red. The gates are cumulative on purpose.
- If Claude Code proposes a design that contradicts `01-FRD.md` or an ADR, the document wins — or you write a new ADR. Never let it silently drift.
- Every phase ends with a green `make gate`, a commit, and a one-line entry in `09-INCIDENT-LOG.md` if anything broke.
- Ship a working vertical slice by end of Day 2 and keep `main` demo-able at every commit after that. A repo that can always demo cannot lose.

---

## 2. Ground rules handed to the agent

These live in `CLAUDE.md` and are restated at the top of every phase context file:

1. **The LLM never executes.** No LLM call may cause a message to send, a charge to attempt, a case to change state, or a policy decision to be made. Violating this fails the phase gate.
2. **No unlogged side effects.** Every state change appends a `CaseEvent`. If you wrote to `cases` without an event, you introduced a bug.
3. **Policy is data, not code paths.** Rules live in `policies/*.yaml`. `if root_cause == "x"` scattered in a service is a defect.
4. **Deterministic first.** Seeded randomness, frozen clocks in tests, no network in unit tests, no wall-clock branching.
5. **Exact money.** `Decimal` and paise-level integers; never `float` for money. UTC internally, IST at the edges only.
6. **Every external call is fallible.** Bounded retry, jitter, timeout, circuit breaker; never mutate state optimistically.
7. **Tests are part of the deliverable, not a follow-up.** Property tests for policy, replay equality for the log, golden sets for the LLM paths.
8. **Type it.** `mypy --strict` on `domain/`, `policy/`, `economics/`, `measurement/`. Pydantic v2 at every boundary.
9. **Honest metrics only.** If a number is not significant, print that it is not significant. Never round a null into a win.
10. **Small commits with meaning.** Conventional Commits, one logical change each, phase trailer on every commit.

---

## 3. Tech stack

Chosen for three properties: **demonstrably serious**, **fast to build solo**, and **free or free-tier**. Each choice carries its justification because "why this stack" is a judging conversation.

### 3.1 Core backend

| Concern | Choice | Why this and not the obvious alternative |
|---|---|---|
| Language | **Python 3.12** | The ML, the stats and the agent tooling all live here; splitting languages for a solo 8-day build is self-harm |
| Package/deps | **uv** + `pyproject.toml` | 10–100× faster installs than pip; reproducible lockfile; CI time matters when you have 8 days |
| API | **FastAPI** + **Pydantic v2** | Schema-first boundaries, automatic OpenAPI (a judge can browse the API), native async, and Pydantic doubles as the LLM structured-output validator |
| ORM / migrations | **SQLAlchemy 2.0** (typed) + **Alembic** | Event store needs precise control over constraints and indexes; Alembic gives reviewable schema history |
| Database | **Postgres 16** | JSONB for event payloads, generated columns, GIN + full-text for grounded retrieval, real transactions for idempotency. One database, no exotic infra |
| Cache / queue | **Redis 7** | Idempotency key set with TTL, rate limits, bandit posteriors, SSE fan-out, arq backend |
| Background work | **arq** | Async-native, Redis-backed, ~200 lines of concept. Celery is heavier; Temporal is the *right* production answer and is documented as the migration path in ADR-0003 — using it here would spend a day on infra instead of on differentiation |
| Scheduling | arq cron + a `due_at` index | Cadence, cooldowns and PTP dates are data, not cron entries |

### 3.2 ML and statistics

| Concern | Choice | Why |
|---|---|---|
| Classification / propensity | **LightGBM** | Fast, strong on tabular, tiny artifacts, trains in seconds so the CI gate is cheap |
| Calibration | **scikit-learn** isotonic / Platt + reliability curves | The EV engine consumes probabilities as real numbers; uncalibrated scores would make the economics fiction |
| Explainability | **SHAP** | Per-case attribution surfaced in the UI and the audit trail |
| Uplift | **X-learner** hand-built over LightGBM (`causalml` optional) | Transparent, debuggable, adequate at this data scale; a causal forest is a heavier dependency for marginal gain |
| Statistics | **statsmodels** + **scipy** | Two-proportion z-test, confidence intervals, MDE, alpha-spending |
| Experiment tracking | Metrics JSON + **model cards** in-repo (MLflow optional) | Artifacts a judge can read in the repo beat a server they can't see |
| Bandit | Hand-rolled Thompson sampling (Beta-Bernoulli per arm) | ~60 lines, fully inspectable, and constrained to policy-permitted arms |

### 3.3 AI layer

| Concern | Choice | Why |
|---|---|---|
| Model | **Claude (Anthropic API)** — Sonnet-class for extraction/drafting, Opus-class for grounded Q&A | Strong structured output and refusal behaviour, which is exactly what the grounded layer needs |
| Structured output | Pydantic schemas + tool-use / JSON mode, with strict validation and a retry-then-degrade path | An invalid parse must degrade to the deterministic path, never block the pipeline |
| Prompt management | Versioned prompt files in `src/recoup/llm/prompts/`, hashed into decisions | A prompt change is a reviewable diff, and old decisions explain with the prompt that produced them |
| Safety | PII redaction before every call; prohibited-claims check on every generated message; cite-or-refuse contract on Q&A | Tested, not asserted |
| Eval | Golden sets in `tests/llm_eval/` run in CI: 40-question grounded benchmark, 60+ PTP utterances, message-safety suite | "Honest metrics" applies to the LLM parts too |
| Build tooling | **Claude Code** with per-phase context files, plan mode before each phase, `make gate` as the acceptance loop | The workflow is itself part of the story: an engineer who directs an agent with specifications and gates, rather than vibes |

### 3.4 Channels and voice

| Concern | Choice | Why |
|---|---|---|
| Channel abstraction | A `ChannelPort` protocol with **simulator-first** adapters | Deterministic, free, offline-demoable; live adapters are a flag, never a dependency (ADR-0006) |
| Simulator | Seeded response model with per-segment open/click/convert curves and realistic latency | Makes the whole system testable and the demo reproducible |
| SMS / voice (live, optional) | **Twilio trial** | Free trial credit is enough for one real call in the pitch video |
| WhatsApp (live, optional) | **Meta WhatsApp Cloud API** sandbox | Free test numbers, real template semantics |
| Email (live, optional) | **Resend** free tier | Trivial integration, real deliverability semantics |
| TTS (Hinglish) | **edge-tts** (`hi-IN` / `en-IN` neural voices), free, offline-friendly | Produces a genuinely convincing Hinglish call artifact at zero cost |
| ASR | **faster-whisper** locally | No per-minute cost, runs on CPU for short utterances |
| Dialogue runtime | Hand-built finite dialogue graph in `src/recoup/voice/` | The bound *is* the feature; a framework would hide the thing being demonstrated |

### 3.5 Frontend

| Concern | Choice | Why |
|---|---|---|
| Framework | **Next.js 15** (App Router) + **TypeScript** | One app serves both the merchant dashboard and the public recovery microsite; deploys free on Vercel |
| UI kit | **Tailwind CSS** + **shadcn/ui** + **lucide** icons | Looks like a real product in hours, not days |
| Charts | **Recharts** (+ a hand-built Qini/calibration chart) | Enough for lift, funnel, calibration and Qini without fighting a viz library |
| Data | **TanStack Query** + **EventSource (SSE)** | Live case stream during the demo is a genuine "this is running" moment |
| Motion | **Framer Motion**, used sparingly | Polish, not decoration |
| Recovery page | Same Next.js app, public route, mobile-first | The judge opens it on their phone |

### 3.6 Platform, quality and delivery

| Concern | Choice | Why |
|---|---|---|
| Local orchestration | **Docker Compose** (Postgres, Redis, API, worker, web) + **Makefile** | `make demo` on a clean machine is the reproducibility claim |
| Lint / format / types | **ruff**, **ruff format**, **mypy** (strict on core) | Cheap signal of discipline |
| Tests | **pytest**, **pytest-asyncio**, **Hypothesis** (property tests), **testcontainers** or a compose fixture for integration | Property tests on the policy engine are the single most persuasive test artifact in the repo |
| Coverage | ≥80% on `policy/`, `economics/`, `measurement/`, `audit/` | Coverage where it matters, not everywhere |
| CI | **GitHub Actions**: lint → types → unit → property → integration → ML metric gate → LLM eval gate → chaos suite | A green badge on a public repo is free credibility |
| Observability | **structlog** JSON logs, **OpenTelemetry** traces, `/metrics` (Prometheus format) | Ingest → decide → act → measure as one trace is a great screenshot |
| Deployment | **Railway** or **Fly.io** (API + worker), **Neon** (Postgres), **Upstash** (Redis), **Vercel** (web) — all free tiers | A live URL in the submission beats a localhost screenshot |
| Secrets | `.env` + `.env.example`, pre-commit secret scan (**gitleaks**) | Never ship a key |
| Pre-commit | ruff, mypy, gitleaks, conventional-commit check | Keeps the history clean without thinking about it |

### 3.7 What we deliberately did **not** use

Saying no is part of the pitch:

- **No LangChain / agent framework.** The authority model is the product; an orchestration framework would obscure exactly the boundary we want to demonstrate.
- **No vector database.** Event logs are short and structured; Postgres FTS plus structured filters retrieves better *and* is auditable (ADR-0005).
- **No Kafka.** One Postgres and one Redis handle four orders of magnitude more than this workload.
- **No microservices.** A modular monolith with clean internal ports; splitting it would cost a day and buy nothing.
- **No Streamlit.** The dashboard is the judge's only direct experience of the product.

---

## 4. Pre-flight checklist (do this before Day 1 starts)

- [ ] Razorpay **test-mode** account; Key ID + Secret; webhook secret generated
- [ ] `ngrok` (or Cloudflare Tunnel) for local webhook delivery
- [ ] Anthropic API key with a spend limit set
- [ ] GitHub account; two **public** repos created: `recoup` and `recoup-docs`
- [ ] Optional free tiers: Twilio trial, Meta WhatsApp sandbox, Resend
- [ ] Free-tier infra accounts: Neon, Upstash, Railway/Fly, Vercel
- [ ] Docker Desktop running; Python 3.12; Node 20+; `uv` installed
- [ ] Screen recorder + mic tested (the pitch video is a deliverable, not an afterthought)

---

## 5. Git strategy

### 5.1 Two repositories, one submodule

```
recoup/                 ← public application repo (the submission)
└── docs/               ← git submodule → recoup-docs (public)
```

**Why a submodule rather than a folder:** the documents are a deliverable with their own review cadence and their own history. A submodule pins the *exact* document revision that the code at any commit was built against — so a reviewer checking out the Day-4 commit gets the Day-4 FRD, not today's. It also lets the docs be reviewed, versioned and even shared independently, and it demonstrates comfort with a tool most submissions won't touch. The cost is one command (`git submodule update --init`), and the README states it in the first three lines.

```bash
# one-time
git submodule add https://github.com/<you>/recoup-docs.git docs
git commit -m "chore(docs): add recoup-docs as submodule"

# after editing documents
cd docs && git add -A && git commit -m "docs: <what changed>" && git push
cd .. && git add docs && git commit -m "chore(docs): bump docs pointer"

# cloning
git clone --recurse-submodules https://github.com/<you>/recoup.git
```

### 5.2 Branches and commits

- `main` is always demo-able. Every phase merges via a short-lived `phase/NN-slug` branch and a self-reviewed PR (PR descriptions become the engineering narrative a judge reads).
- **Conventional Commits**: `feat(policy): enforce quiet hours at evaluation time`.
- Every commit carries a phase trailer:
  ```
  Phase: 04-policy-engine
  Gate: green
  ```
- Tag each completed phase: `git tag phase-04` — the tag list becomes a visible build timeline.

### 5.3 What the repo must contain by Day 8

`README.md` (with the headline numbers and a 60-second quickstart), `docs/` submodule, `context/`, `policies/`, `src/`, `web/`, `ml/` with model cards, `tests/` including property and chaos suites, `Makefile`, `docker-compose.yml`, CI green, a demo GIF, and `docs/09-INCIDENT-LOG.md` filled in honestly.

---

## 6. The eight-day plan

| Day | Phases | Outcome at end of day |
|---|---|---|
| **1** | 00 Foundation, 01 Domain core | Repo, CI, submodule, event store with hash chain, replay test green |
| **2** | 02 Ingestion + synthetic data | Four sources ingest into cases; 500-case seeded batch exists; **first vertical slice demo-able** |
| **3** | 03 Understanding (ML) | Calibrated classifier, propensity, uplift, SHAP, model cards, metric gate in CI |
| **4** | 04 Policy engine, 05 Economics | Policy-as-code + property invariants green; EV gate; approval, staging, kill switch |
| **5** | 06 Execution and channels, 07 Recovery microsite | End-to-end recovery of a case through the simulator; judge-usable recovery page with test-mode checkout |
| **6** | 08 Voice + PTP, 09 Agent and grounded explainability | Hinglish call artifact with a captured promise; grounded Q&A with citations and refusal, CI benchmark green |
| **7** | 10 Measurement, 11 Dashboard + chaos | Incremental ₹ with CI on the dashboard; what-if simulator; "Break it" chaos control working |
| **8** | 12 Submission | Deployed, recorded, documented, submitted |

> Phases are numbered 00–12 below; the day mapping above is where each lands. Each has its own context file in `context/`.

---

### Phase 00 — Foundation *(Day 1, ~3h)*

**Objective:** a repo that is boring, correct, and impossible to make messy later.

**Deliverables:** `recoup` + `recoup-docs` repos; submodule wired; `pyproject.toml` with uv; ruff/mypy/pytest/pre-commit/gitleaks configured; `docker-compose.yml` (Postgres, Redis); `Makefile` targets (`setup data train run demo replay verify chaos gate`); GitHub Actions skeleton; `CLAUDE.md`; `context/` populated; `.env.example`.

**Gate:** `make setup && make gate PHASE=00` green on a clean clone; CI green on first push.

**Prompt seed:**
> Read `CLAUDE.md` and `context/phase-00-foundation.md`. Scaffold the repository exactly as specified in §13 of `docs/03-ARCHITECTURE.md`. Do not write business logic. Produce a Makefile whose `make demo` target is a documented placeholder that fails loudly, and CI that runs lint, types and tests. Show me the plan first.

**Cut line:** none. This phase is not optional and must not exceed half a day.

---

### Phase 01 — Domain core and event store *(Day 1, ~4h)*

**Objective:** the spine — `Case`, `CaseEvent`, the hash chain, projections, replay, idempotency.

**Deliverables:** typed domain models (Pydantic + SQLAlchemy); append-only `case_events` with `prev_hash`/`hash`; canonical JSON serialization; `EventStore.append()` as the *only* write path; projection rebuild (`make replay`); chain verification (`make verify`); idempotency key generation and the Redis + unique-index guard; frozen-clock test utilities.

**Gate:** replay equality test green (projection from log == stored projection); tamper test green (mutating one payload is detected and the divergent event identified); property test: two appends with the same idempotency key produce one effect.

**Prompt seed:**
> Read `context/phase-01-domain-core.md` and §5 of `docs/03-ARCHITECTURE.md`. Implement the event store and domain models. `EventStore.append` must be the only way case state changes — add an architecture test that fails if any module writes to the `cases` table directly.

**Cut line:** none. Everything downstream assumes this.

---

### Phase 02 — Ingestion and synthetic data *(Day 2)*

**Objective:** four sources in, one `Case` out, nothing ever lost — and a realistic 500-case batch to work with.

**Deliverables:** signed Razorpay webhook receiver; dedupe on `(source, provider_event_id)`; normalizers for all four sources; abandonment tracker; mandate poller; receivables CSV importer; DLQ + exception surface; **synthetic data generator** with documented Indian failure-mix distributions, per-case ground-truth response curves, and full seeding.

**Gate:** replaying the same webhook 100× creates one case and zero actions; a malformed payload lands in the DLQ and appears in the exception list; `make data SEED=42` reproduces an identical batch (hash-compared).

**Prompt seed:**
> Read `context/phase-02-ingestion.md`. Implement ingestion for all four sources plus the seeded synthetic generator. The generator must emit a ground-truth `response_curve` per case (probability of self-resolution and probability of resolution under each intervention) so that later phases can validate uplift models against known truth — and that ground truth must never be readable by any model at inference time.

**Cut line:** if time is short, the abandonment tracker can be file-driven rather than live-beacon-driven.

**End of Day 2 milestone: `make demo` shows real cases flowing in. From here, `main` stays demo-able.**

---

### Phase 03 — Understanding: the ML layer *(Day 3)*

**Objective:** diagnosis and targeting that survive a data-science question.

**Deliverables:** LightGBM root-cause classifier with isotonic calibration; per-class metrics, confusion matrix, reliability curve, Brier score; SHAP attribution persisted per case; baseline and treated propensity models; X-learner uplift with Qini evaluation; uplift segmentation; LTV/relationship scorer; model cards; `model_versions` recorded on every decision; CI metric gate.

**Gate:** macro-F1 ≥ 0.85 and Brier ≤ 0.12 on held-out data; positive Qini; a documented list of the classes the model gets wrong and why; CI fails on regression.

**Prompt seed:**
> Read `context/phase-03-understanding.md`. Train the classifier, propensity and uplift models. Report metrics honestly — including a written "known failure modes" section in each model card. Wire the CI metric gate. No model output may bypass calibration.

**Cut line:** if uplift training misbehaves, ship propensity + a documented uplift *heuristic* (self-heal prior subtracted) and label it plainly as a heuristic in the UI. Never fake a model.

---

### Phase 04 — Policy engine *(Day 4, first half)*

**Objective:** the differentiator that judges will probe hardest.

**Deliverables:** `policies/*.yaml` (ladders per root cause, regulatory block, stopping rules, approval thresholds); pure `evaluate()` returning verdict + rule ID + policy version; short-circuit ordering per §7 of the architecture; hot reload + content hashing; **Hypothesis property tests for every invariant**; policy simulator (replay history through a candidate policy); blocked-action reason codes surfaced to the UI.

**Gate:** all invariants green; ≥3 demonstrable blocks (quiet hours, opt-out, revoked-mandate retry) each naming its rule ID; architecture test proving no execution path exists without a logged verdict.

**Prompt seed:**
> Read `context/phase-04-policy-engine.md` and §7 of `docs/03-ARCHITECTURE.md`. Implement the policy engine as a pure function over YAML policy. Write the property-based invariant tests **first**, then make them pass. Any rule that cannot be expressed in YAML must be escalated to me, not hardcoded.

**Cut line:** the policy simulator may slip to Phase 11 (it powers the what-if UI) — the evaluator and invariants may not.

---

### Phase 05 — Economics, approval and reversibility *(Day 4, second half)*

**Objective:** the system learns to say "not worth it" and "ask a human first".

**Deliverables:** channel cost model; goodwill cost curve; EV engine with per-action ledger; EV floor and `abandoned_uneconomic` terminal state; per-customer rolling contact-fatigue budget; merchant daily caps; approval queue with the decision card payload; **staged-action buffer with an undo window**; exposure cap; global kill switch; override auditing.

**Gate:** a case terminates `abandoned_uneconomic` with its arithmetic visible; a staged action is cancelled inside its window and never sends; the kill switch cancels everything in flight and is logged with an actor.

**Prompt seed:**
> Read `context/phase-05-economics-and-authority.md`. Implement EV gating and the staged-action buffer. Nothing may leave the system without passing through a cancellable state — add a test that asserts every executed action was `staged` first.

**Cut line:** none. This is the "would I put you near money" phase.

---

### Phase 06 — Execution and channels *(Day 5, first half)*

**Objective:** actions actually happen — safely, idempotently, and for free.

**Deliverables:** `ChannelPort` protocol; seeded deterministic simulator with per-segment response curves; optional live adapters (Twilio/WhatsApp/Resend) behind a flag; message rendering from templates with Claude-drafted copy (schema-validated, PII-redacted, prohibited-claims checked); delivery-state events; constrained Thompson-sampling bandit over permitted (channel, hour) arms; channel-fatigue suppression; retry/backoff/circuit breakers.

**Gate:** a treatment case moves ingest → diagnose → score → EV → policy → stage → send → engage → recover end-to-end in the simulator; the bandit provably never selects a policy-denied arm (property test).

**Cut line:** live adapters are optional; the bandit can degrade to channel-fit + fixed timing if it misbehaves (documented as such).

---

### Phase 07 — Self-serve recovery microsite *(Day 5, second half)*

**Objective:** the thing a judge touches with their own thumb.

**Deliverables:** signed single-use expiring links (HMAC + nonce + TTL); public Next.js route rendering a context-aware page per root cause; correct fix per cause (retry / switch method / update card / re-authorize mandate / pay invoice) via Razorpay test-mode Payment Links or Checkout; opt-out and "remind me later" honoured by the policy engine; page events fed back as case events; mobile-first and fast.

**Gate:** on a phone, open the link from a simulated SMS, complete a test-mode payment, and watch the dashboard case flip to `recovered` over SSE within seconds. Link reuse is refused.

**Cut line:** if the checkout integration fights back, fall back to Payment Links (simpler, hosted) — do not lose the live-payment moment.

---

### Phase 08 — Hinglish voice and promise-to-pay *(Day 6, first half)*

**Objective:** the thirty seconds of the pitch video that nobody else has.

**Deliverables:** finite dialogue graph (identify → disclose → purpose → offer → capture/objection → confirm → close) with explicit exits; edge-tts Hinglish synthesis; faster-whisper ASR; safe degradation on low confidence, silence, hostility, or out-of-graph input; disclosure + opt-out + human-transfer honoured in-call; transcript, audio artifact and node path attached to the case and hash-chained; PTP extraction into a strict schema with a confidence threshold; PTP suspends escalation until the committed date + grace; trust score feedback; 60+ utterance golden set with precision/recall.

**Gate:** a recorded call in which a promise is captured, escalation visibly goes quiet until the promised date, and an out-of-graph utterance degrades gracefully rather than improvising.

**Cut line:** if live telephony is fragile, render the call **offline as an audio artifact** against the same dialogue graph. The graph and the captured promise are the substance; live telephony is garnish.

---

### Phase 09 — Grounded explainability *(Day 6, second half)*

**Objective:** the chat layer that refuses to lie.

**Deliverables:** retrieval over `case_events` (Postgres FTS + structured filters); answer composition that cites event IDs and renders them as clickable links to the timeline; hard refusal contract when the log lacks the answer, naming what would need to be logged; PII redaction before every call; latency budget with deterministic fallback; **40-question CI benchmark** (20 answerable, 20 unanswerable) failing the build on a fabricated citation or a missed refusal.

**Gate:** benchmark green; live demo of one answered question with citations and one refused question.

**Cut line:** none — this is a headline differentiator. Reduce the benchmark to 20 questions before dropping the refusal contract.

---

### Phase 10 — Measurement *(Day 7, first half)*

**Objective:** the number the whole submission rests on.

**Deliverables:** stratified seeded cohort assignment at case creation with an immutability test; control-integrity invariant (zero actions on control); adaptive holdout controller with alpha-spending; incremental recovery in ₹ and %; two-proportion z-test, 95% CI and MDE; CUPED adjustment with the unadjusted number always shown; breakdowns by cause, channel, segment and value band; raw-vs-incremental comparison; batch report export (JSON + Markdown/PDF).

**Gate:** `make demo` prints the headline block — raw ₹, incremental ₹, lift %, CI, p-value, MDE, cost per ₹ recovered — and it is byte-identical on a second run from the same seed.

**Prompt seed:**
> Read `context/phase-10-measurement.md` and `docs/05-EVALUATION-PROTOCOL.md`. Implement the measurement engine exactly as specified. If the lift is not significant at the batch size run, the output must say so explicitly. Do not add any code path that could make a non-significant result look significant.

**Cut line:** CUPED and the adaptive controller can degrade to a fixed 20% holdout with a plain z-test. The control group itself is non-negotiable.

---

### Phase 11 — Dashboard, what-if and chaos *(Day 7, second half)*

**Objective:** make every differentiator visible in five seconds, and break the system on purpose.

**Deliverables:** Next.js dashboard — batch summary with raw vs. incremental, ranked work queue with reasons, approval queue with decision cards and working undo, exception/DLQ view, case timeline from the event log, compliance view of blocked actions with rule IDs, model transparency panel (confusion matrix, calibration, Qini), grounded Q&A panel, live SSE stream; **what-if simulator** replaying history through modified policy; **"Break it" chaos control** running the chaos suite live.

**Gate:** each of the 10 acceptance criteria in `01-FRD.md` §14 is reachable in ≤2 clicks from the dashboard. Chaos scenarios visibly produce zero duplicate actions and truthful exceptions.

**Cut line:** the what-if simulator is the first UI feature to drop; the compliance view and the case timeline are not.

---

### Phase 12 — Submission *(Day 8)*

**Objective:** convert a working system into an accepted, memorable submission.

**Deliverables:**

- Deploy: API + worker (Railway/Fly), Postgres (Neon), Redis (Upstash), web (Vercel). Seeded demo data loaded. Public URL in the README.
- `README.md`: one-paragraph pitch, the headline numbers with their CI, a 60-second quickstart, an architecture diagram, a link to the live demo, a demo GIF, and the "what we deliberately did not build" section.
- **5-minute pitch video** per `07-DEMO-SCRIPT.md`.
- `09-INCIDENT-LOG.md` completed honestly — this is the artifact that answers *"what broke and how you got out"*.
- Final CI green; all phase tags pushed; submodule pointer bumped; repos public.
- Submit on razorpay.com/buildathon with the repo, video and architecture doc links.

**Gate:** a stranger clones the repo, runs `make demo`, and sees the same headline numbers. Verify this on a clean machine or a fresh container — do not assume it.

---

## 7. Descope ladder

When (not if) time runs short, cut in this exact order. Never improvise the order under pressure:

1. What-if simulator UI (keep the policy simulator in code)
2. Live channel adapters (simulator remains)
3. Contextual bandit (fall back to channel-fit + fixed timing)
4. CUPED and adaptive holdout (fixed 20% holdout + z-test remains)
5. Live telephony (offline Hinglish audio artifact remains)
6. SHAP in the UI (keep it in the audit trail)
7. OpenTelemetry traces (structured logs remain)

**Never cut:** the control group, the policy engine and its invariant tests, idempotency, the hash-chained log with replay, the approval gate and staged undo, the grounded-refusal contract, the chaos suite, or the honest reporting of a non-significant result.

---

## 8. Daily rhythm

**Morning (15 min):** re-read the phase context file; write the day's cut line in `09-INCIDENT-LOG.md`; check yesterday's gate is still green.

**Build blocks:** one phase per Claude Code session, plan mode first, gate at the end, commit and tag.

**Evening (20 min):** run `make demo` end-to-end — every day, from Day 2. Record what broke. Push. If `main` is not demo-able, fix that before sleeping; nothing else matters more.

---

## 9. Risk register

| Risk | Trigger | Response |
|---|---|---|
| Razorpay webhooks unreliable locally | Tunnel drops, events missed | Recorded-fixture replay path (`make replay-fixtures`) built in Phase 02; never demo live-dependent |
| ML metrics below the gate | Day 3 evening | Simplify features, accept a lower documented bar, and report it honestly — a stated 0.79 macro-F1 beats a fabricated 0.92 |
| Uplift model unstable on small data | Qini near zero | Ship the documented heuristic, label it as such, and explain the production path |
| Voice consumes Day 6 entirely | Nothing else ships Day 6 | Hard 4-hour cap; fall back to the offline audio artifact |
| Deployment eats Day 8 | Free-tier friction | Deploy a skeleton on Day 5 evening so Day 8 is a redeploy, not a first attempt |
| Demo fails live | Provider or network issue | Everything runs on the simulator by default; record a backup demo video on Day 7 |
| Scope creep from good ideas | Any day | New ideas go into `docs/BACKLOG.md`, never into the current phase |

---

## 10. Submission definition of done

- [ ] Public repo, CI green, submodule pointer current, phase tags pushed
- [ ] `make demo` reproduces the headline numbers from a seed on a clean machine
- [ ] All 14 acceptance criteria in `01-FRD.md` §14 demonstrable
- [ ] Live deployed URL working with seeded demo data
- [ ] 5-minute pitch video uploaded and linked
- [ ] Architecture documentation (`03-ARCHITECTURE.md`) linked in the README
- [ ] Incident log completed honestly
- [ ] Headline metrics stated **with** their confidence interval, and the raw number shown beside the incremental one
- [ ] Submitted before the deadline with at least a day's margin
