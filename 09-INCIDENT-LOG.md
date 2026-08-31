# Recoup — Build Incident Log

**What broke, how it was diagnosed, and how it was fixed.**

> The buildathon explicitly asks *"what broke and how you got out"*. This file is the answer, written as it happens rather than reconstructed afterwards. An empty incident log on an 8-day build is not evidence of a smooth build; it is evidence of a log nobody kept.

## How to use this

Add an entry the moment something costs you more than 15 minutes. Write it before you fix it if you can — the diagnosis is the interesting part, and it evaporates once the bug is gone. Keep entries short and specific. Do not sanitize them.

**Entry template:**

```markdown
### INC-00N · <one-line symptom>

- **When:** Day N, phase NN
- **Symptom:** what you actually observed, not what you assumed
- **Blast radius:** what was broken, what still worked
- **First wrong hypothesis:** what you suspected and why it was wrong (keep this — it is the most honest part)
- **Diagnosis:** how you found the real cause (the command, the log line, the test that isolated it)
- **Root cause:** the actual mechanism
- **Fix:** what changed
- **Prevention:** the test, invariant, or check that now makes this class of bug fail loudly
- **Time lost:** Xh
```

---

## Entries

### INC-001 · `make setup` couldn't reach Postgres or Redis on this dev machine

- **When:** Day 1, phase 00
- **Symptom:** `docker compose up -d postgres redis` failed with "port is already allocated" on 6379; after remapping Redis, `alembic upgrade head` failed with `InvalidPasswordError` for a user/password that work fine via `docker exec psql`.
- **Blast radius:** local environment only; nothing in the repo's logic was wrong.
- **First wrong hypothesis:** assumed the Postgres credentials in `.env` didn't match `docker-compose.yml`'s `POSTGRES_PASSWORD`. They matched.
- **Diagnosis:** `netstat -ano | grep :5432` showed a *second*, unrelated process already listening on 5432 — a native Windows Postgres service from another local project. Docker's own container was healthy and had the right credentials; TCP connections to `localhost:5432` were landing on the wrong Postgres entirely. Same root cause explained the earlier Redis port clash (an unrelated project's `electra-redis-1` already held 6379).
- **Root cause:** this dev machine runs other local projects with long-lived Postgres/Redis on the default ports; Recoup's compose file assumed a clean machine.
- **Fix:** added a git-ignored `docker-compose.override.yml` that remaps the host side only (`5434:5432`, `6380:6379`) via the Compose `!override` merge tag, and pointed the local (git-ignored) `.env` at the remapped ports. `docker-compose.yml` itself — the file a clean machine or CI actually uses — is untouched.
- **Prevention:** none needed in the codebase; this is inherent to running multiple projects' default-port datastores side by side. Documented here so a future session doesn't re-diagnose it from scratch.
- **Time lost:** 0.3h

### INC-002 · `scripts/phase_gate.py` crashed on Windows before it ever printed a gate result

- **When:** Day 1, phase 00
- **Symptom:** `make gate PHASE=00` got through lint, types and the full test suite, then crashed with `UnicodeEncodeError` on the final summary line.
- **Blast radius:** cosmetic only — every real check had already passed — but a script that crashes on its own success line makes a green gate look red.
- **First wrong hypothesis:** none; the traceback pointed straight at the box-drawing character in the f-string.
- **Diagnosis:** the traceback showed `cp1252` (Windows' default console codepage) rejecting a Unicode dash the script printed; the same characters had already shown up as `�` in the CLI's placeholder messages.
- **Root cause:** Python's stdout encoding on Windows defaults to the console codepage, not UTF-8, unlike Linux CI.
- **Fix:** `sys.stdout.reconfigure(encoding="utf-8")` in `scripts/phase_gate.py`; replaced the Unicode dashes in `cli.py`'s placeholder messages with plain ASCII hyphens so the CLI degrades safely without relying on any particular console encoding.
- **Prevention:** none automated; noted here since later phases will add more CLI/script output and should default to ASCII-safe text or reconfigure stdout up front rather than discover this per-script.
- **Time lost:** 0.2h

### INC-000 · (template example — replace with the first real incident)

- **When:** Day 2, phase 02
- **Symptom:** replaying the same Razorpay webhook fixture produced two cases, but only on the second run of the test suite
- **Blast radius:** ingestion only; nothing downstream had run yet
- **First wrong hypothesis:** assumed the dedupe index was missing. It was present.
- **Diagnosis:** `SELECT source, provider_event_id, count(*) ... GROUP BY 1,2 HAVING count(*) > 1` showed the same event ID stored with different whitespace in the JSON payload — the dedupe key was being derived from a hash of the serialized payload rather than from the provider event ID.
- **Root cause:** non-canonical JSON serialization made a logically identical payload hash differently.
- **Fix:** dedupe on `(source, provider_event_id)` explicitly; canonical JSON (sorted keys, no whitespace) for all hashing, extracted into `domain/canonical.py` and reused by the audit hash chain.
- **Prevention:** property test asserting `hash(canonical(x)) == hash(canonical(reordered(x)))`, plus a duplicate-webhook scenario in the chaos suite.
- **Time lost:** 1.2h

---

## Running summary (fill in at the end, this is what the judges read)

| # | Phase | One-line summary | Time lost | Prevention now in place |
|---|---|---|---|---|
| 001 | 00 | Two other local projects already held ports 5432/6379 on this dev machine | 0.3h | git-ignored `docker-compose.override.yml` remaps host ports locally; `docker-compose.yml` stays clean-machine-correct |
| 002 | 00 | `phase_gate.py` crashed on Windows' console codepage before printing its own result | 0.2h | UTF-8 stdout in the gate script; ASCII-only CLI placeholder text |

**Total time lost to incidents:** 0.5h
**Most valuable lesson:** _
**The bug that would have shipped if it hadn't been caught:** _

---

## Standing reflection prompts (answer these on Day 8)

1. Which failure would have been invisible without the audit log?
2. Which invariant test caught something a manual test would have missed?
3. Where did an AI coding agent confidently produce something subtly wrong, and how did the specification catch it?
4. What did you deliberately choose *not* to fix, and why was that the right call under the deadline?
