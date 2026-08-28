# ADR-0003 · Postgres + Redis + arq instead of Temporal

**Status:** Accepted · Phase 00

## Context
Recovery is a long-running, multi-step workflow with waits, cooldowns and compensations — textbook Temporal territory. Temporal would give durable execution and retries for free.

## Decision
Use a state machine over the event log, with `arq` workers and a `due_at` index for waking cases. Do not run Temporal in v1.

## Consequences
- **Positive:** one less service to operate, deploy and explain; the state machine is inspectable in the same log that powers the audit trail; a day of infra time goes into differentiation instead.
- **Negative:** we hand-roll retry, lease and timeout semantics. Bounded by the fact that every action is idempotent and staged.
- **Migration path:** documented explicitly. When cadence logic outgrows the state machine, Temporal is the answer, and the event log makes the migration mechanical.
