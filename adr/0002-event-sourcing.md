# ADR-0002 · Event sourcing with a hash chain

**Status:** Accepted · Phase 01

## Context
The track bar requires an audit trail. The cheap version is a `logs` table written alongside a mutable `cases` table — which cannot prove that state never changed without a log write, cannot reconstruct past state, and cannot replay history through a different policy.

## Decision
`case_events` is append-only and is the source of truth. `cases` is a projection rebuilt by `make replay`. Each event stores `prev_hash` and `hash = sha256(prev_hash ‖ canonical_json(payload) ‖ seq ‖ occurred_at)`.

## Consequences
- **Positive:** time travel, tamper evidence, the what-if policy simulator, and post-hoc compliance replay all fall out of this one choice. Replay equality is a CI test.
- **Negative:** more code than CRUD; every write path must go through `EventStore.append`; projections can drift if the rebuild is not tested. Mitigated by an architecture test that fails on any direct write to `cases`.
