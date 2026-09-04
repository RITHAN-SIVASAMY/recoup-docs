# Recoup — Submission Checklist

**Track 03 · AI Revenue Recovery · Razorpay AI Buildathon**

> Deadline per public announcements: **5 September 2026**. Plan to submit on **Day 8 with a full day of margin**. Verify the deadline and submission mechanics directly on razorpay.com/buildathon before relying on this line.

---

## 1. What the organizers ask for

| Requirement | Our artifact | Status |
|---|---|---|
| Public code repository with clean documentation | `github.com/RITHAN-SIVASAMY/recoup` (+ `recoup-docs` submodule) | ☑ both public |
| 5-minute pitch video showing the working product | Per `07-DEMO-SCRIPT.md` | ☐ |
| Architecture documentation | `docs/03-ARCHITECTURE.md`, linked from the README | ☑ |
| Resume | Attach; keep it one page | ☐ |
| *"What broke and how you got out"* | `docs/09-INCIDENT-LOG.md` — a real artifact, not an anecdote | ☑ 27 incidents, honest |

---

## 2. Track bar, line by line

The bar: *"Don't just identify the problem. Show measured money recovered across a batch, with compliant escalation, stopping rules, and an audit trail."*

| Phrase | Where a judge sees it satisfied | Status |
|---|---|---|
| **measured** money recovered | Incremental ₹ with 95% CI, z-test and MDE, raw number shown beside it | ☑ |
| across a **batch** | 2,000 seeded cases, reproducible with `make demo` (raised from 500 — INC-023: 500 was honestly under-powered for this data's effect size) | ☑ |
| **compliant** escalation | Compliance view: blocked actions with rule IDs; `06-COMPLIANCE-MATRIX.md` | ☑ |
| **stopping rules** | Policy-as-code + Hypothesis invariant tests; ≥3 blocks demonstrated live | ☑ (388 blocked in the canonical batch alone) |
| **audit trail** | Hash-chained event log, deterministic replay, grounded Q&A with citations and refusal | ☑ (`make verify`: 25,703 events, chain verified, replay equality PASS) |
| detects revenue at risk | Four sources ingesting into one case; DLQ for the rest | ☑ |
| determines the right intervention | Cause-specific ladders, uplift, EV gate, constrained bandit | ☑ |
| executes a **bounded** workflow | Approval gate, staged undo, exposure cap, kill switch | ☑ |

---

## 3. Repository

- [x] `README.md` opens with one paragraph, the headline numbers **with CI**, and a 60-second quickstart
- [x] `git clone --recurse-submodules` documented in the first three lines
- [x] `make setup && make demo` works on a clean machine — CI runs the equivalent steps (fresh checkout, `alembic upgrade head`, train, full suite) on a genuinely fresh `ubuntu-latest` runner and passes; `docker compose up` additionally verified end to end tonight (INC-027) after finding it had never once worked
- [x] CI green; badge in the README — all 4 jobs (`quality`/`tests`/`models`/`llm-eval`) confirmed green on `main`
- [x] `docs/` submodule pointer bumped to the final revision
- [ ] Phase tags `phase-00` … `phase-12` pushed — `phase-00`–`phase-11` are up; `phase-12` is deliberately not tagged until the video is recorded and the submission is actually in, not before
- [x] `policies/*.yaml` readable by a non-engineer
- [x] `ml/` contains model cards with **Known failure modes** sections — `ml/cards/{classifier,propensity,uplift}.md`, each volunteering real numbers and real limitations
- [x] `tests/` includes property, chaos and LLM-eval suites, and they pass — unit 37 · property 12 · integration 24 · chaos 2 · llm_eval 3 files, 423 passed in CI
- [ ] Demo GIF in the README — three real static screenshots are in (Overview, Insights, a live chaos result); an animated GIF still needs a screen recording, which only the presenter can produce
- [x] Architecture diagram renders on GitHub (Mermaid) — verified rendering live on the public repo
- [x] No secrets committed; `gitleaks` clean; `.env.example` complete — gitleaks passing in CI (v3, after INC-026); `.env.example` matches `Settings` field-for-field
- [x] LICENSE (MIT) and a short CONTRIBUTING/AGENTS note explaining the Claude Code workflow — `AGENTS.md`

---

## 4. Live deployment

**Deliberately cut**, not silently missing — see `docs/09-INCIDENT-LOG.md`'s standing
reflection on why: a half-configured deployment that breaks during judging is worse than an
honestly-labeled "runs locally" repo with a genuinely verified quickstart. All boxes below
stay unchecked on purpose.

- [ ] API + worker deployed (Railway/Fly), health endpoint public
- [ ] Postgres (Neon) and Redis (Upstash) provisioned; seeded demo batch loaded
- [ ] Dashboard live on Vercel; URL in the README
- [ ] Recovery microsite reachable on mobile; test-mode payment completes end to end
- [ ] Demo credentials (read-only) in the README so a judge can log in without asking
- [ ] Rate limits and a spend cap on the Anthropic key — a public demo is a public spend surface
- [ ] A banner stating **test mode / synthetic data** on every public page

---

## 5. Video

- [ ] ≤ 5:00, 1080p, audible, captions on key numbers
- [ ] Every claim in the video is visible on screen while it is spoken
- [ ] The honest-limitation line is included, near the end
- [ ] Uploaded unlisted with link sharing on; link tested in an incognito window
- [ ] Backup take recorded on Day 7

---

## 6. Final self-review — the five questions a judge will ask

Rehearse a 30-second answer to each. If any answer is weak, that is the last thing to fix.

1. **"How do you know they wouldn't have paid anyway?"** → Randomized stratified control, z-test, CI, MDE; adaptive holdout with a logged schedule.
2. **"What stops this from spamming customers?"** → Policy-as-code with property-tested invariants; quiet hours, opt-out, rolling fatigue budget, hard caps; ≥3 blocks demonstrated live.
3. **"What happens when it's wrong or something breaks?"** → EV gate refuses; low-confidence routes to a human; chaos suite on stage; exception queue; degraded mode; nothing lost.
4. **"How much of this is the LLM?"** → The authority table in `01-FRD.md` §11. The LLM drafts, extracts and explains. It never decides or executes. Point at the architecture test that enforces it.
5. **"What would you do with more time?"** → `03-ARCHITECTURE.md` §14: Temporal, a real uplift feedback loop, per-merchant models, a policy DSL with proof obligations. Knowing the limits is part of the answer.

---

## 7. Do not do this

- Do not report a raw recovery number as the headline.
- Do not claim production performance from synthetic data.
- Do not claim regulatory certification. Alignment by design, verification required — say it exactly that way.
- Do not demo anything that depends on live third-party providers.
- Do not exceed five minutes.
- Do not hide the failure modes. They are the most credible thing in the submission.
