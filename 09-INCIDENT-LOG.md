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

### INC-008 · Two tests used fixed literal seeds against a persistent shared database

- **When:** Day 2, phase 02
- **Symptom:** `test_generator_ingest.py` passed on its first run and failed on the second (`assert result.created is True` / `assert all(first_run)`) with no code change in between.
- **Blast radius:** those two tests only; not a real defect in the generator or `ingest()`.
- **First wrong hypothesis:** none — this was recognized immediately as the same shape as the `pay_replay_test_001` literal earlier in `test_webhook.py` (never fully written up at the time; folding the lesson in here instead).
- **Diagnosis:** both tests called `generate_batch(seed=999_042, ...)` / `seed=999_043`, and `generate_batch` is deliberately deterministic — so the *second* run of the test file ingested exactly the same `provider_event_id`s the *first* run had already committed, and dedupe correctly reported them all as duplicates.
- **Root cause:** confusing "the generator must be deterministic" with "tests may hardcode a seed" — the dev database persists across test runs, so any test that asserts "this is the first time this data has been seen" needs data that's actually novel each run, not merely reproducible.
- **Fix:** both tests now draw their seed from `random.SystemRandom()` at run time — the generator is still exercised deterministically *within* one call, but the test itself is idempotent across repeated runs.
- **Prevention:** general pattern, not a one-off: any integration test asserting first-time creation needs either a fresh unique id per run (as `tests/factories.py` and most of this suite already does) or, for the generator specifically, a randomly-drawn seed.
- **Time lost:** 0.2h

### INC-007 · A genuine dedupe race: the reservation can outrun the case it reserved

- **When:** Day 2, phase 02
- **Symptom:** a chaos test firing 10 truly concurrent first-deliveries of the same `provider_event_id` (via `asyncio.gather`, not a sequential loop) crashed one of the 10 with `ValueError: case ... does not exist; the first event for a case must be case.created, got 'event.duplicate_suppressed'`.
- **Blast radius:** only genuinely simultaneous first deliveries of the same event — the far more common sequential-replay case (already covered by the 100x tests) was never affected.
- **First wrong hypothesis:** none — the chaos test was written specifically to probe this path, and it found a real bug on the first run.
- **Diagnosis:** `ingest()`'s dedupe reservation (`provider_events`, its own atomic insert-or-read) and the actual case creation (`EventStore.append`, a separate transaction) are two different resources. The race winner reserves the slot, commits, and *then* calls `EventStore.append(case.created)` — there's a real window where a concurrent loser reads the winning `case_id` from `provider_events` before the winner's `case.created` event has landed.
- **Root cause:** treating "reserved the dedupe slot" and "the case exists" as the same moment, when they're two separate commits.
- **Fix:** losers now wait (bounded poll, 20 × 50ms) for `EventStore.events_for(case_id)` to become non-empty before appending `event.duplicate_suppressed`, instead of racing the invariant. A `TimeoutError` if the winner never shows up is the honest failure mode, not a silent skip.
- **Prevention:** `tests/chaos/test_ingestion_chaos.py::test_concurrent_first_deliveries_of_the_same_event_still_produce_one_case` — the same scenario, kept in the chaos suite so a future refactor that reintroduces this race fails loudly again.
- **Time lost:** 0.3h

### INC-006 · `TestClient` + asyncpg is unusable on Windows; `httpx.AsyncClient` isn't

- **When:** Day 2, phase 02
- **Symptom:** every webhook integration test failed on its first DB write with `asyncpg.exceptions.InterfaceError: cannot perform operation: another operation is in progress`, even the very first test in a clean run.
- **Blast radius:** the webhook route's own test coverage only; the route code itself was fine.
- **First wrong hypothesis:** suspected the app's process-cached `get_engine()` (an `lru_cache`'d singleton, correct for a real single-loop server process) was being reused across two different event loops within the test run and corrupting its connection pool.
- **Diagnosis:** a minimal repro (one FastAPI route, one async engine, two `TestClient` calls) reproduced it standalone. The real traceback bottomed out in `asyncio.proactor_events.py`: `self._loop._proactor` was `None` — Starlette's `TestClient` drives the ASGI app from a separate worker-thread event loop (an anyio `BlockingPortal`), and handing an asyncpg connection across that thread/loop boundary breaks outright on Windows's `ProactorEventLoop`.
- **Root cause:** `TestClient` + asyncpg + Windows is a real, documented incompatibility — not a bug in this codebase.
- **Fix:** rewrote the webhook integration tests on `httpx.AsyncClient(transport=ASGITransport(app=app))` inside `async def` tests, so the whole request (including its DB calls) runs on the same loop as the test itself; also overrode the app's `get_session`/`get_event_store` dependencies with a per-test `engine` fixture rather than the process-cached one, so nothing here depends on which loop created which connection.
- **Prevention:** none needed beyond the pattern itself — documented here so a later phase's API tests default to `AsyncClient`/`ASGITransport` on this project rather than rediscovering the same wall.
- **Time lost:** 0.6h

### INC-005 · The socket-blocking guard broke every async unit test on Windows

- **When:** Day 2, phase 02
- **Symptom:** the first `async def` unit test added in this phase (`test_mandate_poller.py`, pure in-memory, no real I/O) failed at fixture setup with `pytest_socket.SocketBlockedError`, not inside the test body.
- **Blast radius:** any future async unit test; nothing that had already shipped, since Phase 01's unit tests were all synchronous.
- **First wrong hypothesis:** assumed the test itself was accidentally touching the network.
- **Diagnosis:** the traceback was entirely inside `pytest_asyncio`/`asyncio.windows_events`: creating a fresh event loop calls `_make_self_pipe`, which calls `socket.socketpair()` — a real socket, even for a loop that will never do network I/O. `tests/conftest.py`'s `pytest_runtest_setup` hook disabled sockets before pytest-asyncio's own loop-creation fixture ran, so the loop itself couldn't be built.
- **Root cause:** `ProactorEventLoop` (Windows' default) needs a real socket for internal plumbing; the guard was disabling sockets too early in the test lifecycle to allow that.
- **Fix:** moved the disable/enable to wrap only `pytest_runtest_call` (the test body) via a hookwrapper, leaving fixture setup/teardown — where the loop gets created and destroyed — with sockets enabled throughout.
- **Prevention:** this is the general form of the fix, not a one-off patch, so it should hold for every async unit/property test from here on.
- **Time lost:** 0.3h

### INC-004 · `Decimal('2500')` vs `Decimal('2500.00')` broke replay equality — a real bug

- **When:** Day 2, phase 02
- **Symptom:** `verify_replay_equality` failed again after the webhook route started working, this time for a different reason: the rebuilt `Case.amount_at_risk` and the stored one were numerically equal but compared unequal.
- **Blast radius:** every case created through `payment_failed.normalize()` (paise / 100 division), none created directly with a pre-quantized amount.
- **First wrong hypothesis:** assumed this was another instance of INC-003 (stale pre-schema-change event data) and almost reset the database without looking closer.
- **Diagnosis:** a small script projected the specific diverging case and printed both sides: `amount_at_risk=Decimal('2500')` (rebuilt from the event payload) vs `Decimal('2500.00')` (read back from the `NUMERIC(14,2)` column). `Decimal('2500') == Decimal('2500.00')` is `True` in Python — but `str()` of each differs, and the replay-equality check compares canonical JSON strings, not numeric Decimal equality, by design (that's what "byte-for-byte" means).
- **Root cause:** `Decimal(250000) / Decimal(100)` drops trailing zeros (`Decimal('2500')`, not `Decimal('2500.00')`); nothing quantized it before it was stringified into the `case.created` payload, while Postgres always returns `NUMERIC(14,2)` at exactly 2 decimal places.
- **Fix:** a pydantic `field_validator` on `NormalizedIntake.amount_at_risk` quantizes to 2dp (`ROUND_HALF_UP`) at construction time, so every code path that reads it — the payload, the projection, the DB column — agrees on the same string form.
- **Prevention:** the quantization happens at the one place `Decimal` amounts enter the system from raw provider data, not scattered at each call site. Reset the dev database once more (same justification as INC-003) since old events still carried the unquantized form.
- **Time lost:** 0.4h

### INC-003 · Extending `Case` broke replay equality against the dev database's own history

- **When:** Day 2, phase 02
- **Symptom:** `verify_replay_equality` started failing with `KeyError: 'merchant_id'` after `Case` gained `merchant_id`/`provider_event_id`/`amount_at_risk`/`currency`/`customer_ref` (needed for ingestion's FR-1.5 normalization) — but only in the integration suite, never in unit tests.
- **Blast radius:** none real; the dev Postgres database.
- **First wrong hypothesis:** suspected a bug in the new `fold()` case.created handler itself.
- **Diagnosis:** `verify_replay_equality`/`verify_chain` scan *every* event ever committed to the shared dev database — including Phase 01's own test runs, which wrote `case.created` payloads shaped like `{"source_type": "..."}` only, before the new required fields existed. `fold()` correctly refused to project them.
- **Root cause:** this is the real event-sourcing schema-evolution problem — an old event's payload shape and the current `fold()` logic disagreed. In production this needs versioned event schemas and upcasting; for a pre-launch dev database with zero real data, it does not.
- **Fix:** `docker compose down -v` on the `recoup` project only (verified it does not touch the unrelated project's containers on this machine) and re-ran `alembic upgrade head` for a clean slate. No code change; this is not a pattern to reach for once real data exists.
- **Prevention:** none added — deliberately. The real prevention (event schema versioning) is out of scope until it's needed; noting it here so a future phase doesn't rediscover the same tradeoff from scratch.
- **Time lost:** 0.2h

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
| 003 | 02 | Extending `Case`'s columns broke replay equality against Phase 01's own old test events | 0.2h | none needed — pre-launch dev-DB reset; noted as the general event-schema-evolution tradeoff |
| 004 | 02 | `Decimal('2500')` vs `Decimal('2500.00')` — a real quantization bug, not stale data | 0.4h | `NormalizedIntake.amount_at_risk` quantizes to 2dp at construction, once, for every caller |
| 005 | 02 | Socket-blocking guard disabled sockets before pytest-asyncio could create its own loop on Windows | 0.3h | guard now wraps only the test body, not fixture setup/teardown |
| 006 | 02 | `TestClient` + asyncpg is broken on Windows (cross-thread event loop handoff) | 0.6h | webhook tests use `httpx.AsyncClient` + `ASGITransport` on the test's own loop instead |
| 007 | 02 | Real race: dedupe reservation could outrun the case.created it pointed to | 0.3h | losers wait for the case to exist before logging the suppression; covered by a chaos test |
| 008 | 02 | Two tests hardcoded a seed against a persistent shared dev database | 0.2h | seeds drawn from OS entropy at test-run time, not literals |

**Total time lost to incidents:** 2.5h
**Most valuable lesson:** _
**The bug that would have shipped if it hadn't been caught:** _

---

## Standing reflection prompts (answer these on Day 8)

1. Which failure would have been invisible without the audit log?
2. Which invariant test caught something a manual test would have missed?
3. Where did an AI coding agent confidently produce something subtly wrong, and how did the specification catch it?
4. What did you deliberately choose *not* to fix, and why was that the right call under the deadline?
