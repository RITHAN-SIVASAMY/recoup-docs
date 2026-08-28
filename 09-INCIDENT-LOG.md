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

<!-- Add entries here as they happen. Examples of the kind of thing worth logging,
     drawn from the failure modes this architecture is most likely to hit: -->

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
| | | | | |

**Total time lost to incidents:** _h
**Most valuable lesson:** _
**The bug that would have shipped if it hadn't been caught:** _

---

## Standing reflection prompts (answer these on Day 8)

1. Which failure would have been invisible without the audit log?
2. Which invariant test caught something a manual test would have missed?
3. Where did an AI coding agent confidently produce something subtly wrong, and how did the specification catch it?
4. What did you deliberately choose *not* to fix, and why was that the right call under the deadline?
