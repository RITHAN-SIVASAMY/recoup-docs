# ADR-0005 · Postgres full-text search for grounded retrieval

**Status:** Accepted · Phase 09

## Context
The grounded Q&A layer retrieves audit-log entries to answer questions with citations. The reflexive choice is embeddings plus a vector database.

## Decision
Retrieve with Postgres full-text search combined with structured filters (case, actor, event type, time range). Add `pgvector` only if lexical retrieval measurably fails on the benchmark.

## Consequences
- **Positive:** no extra service; retrieval is deterministic and explainable (the filter is visible); structured filters dominate at this scale because the questions are inherently scoped to a case; citations map to real primary keys.
- **Negative:** weaker on paraphrase. Mitigated by query expansion over the event-type vocabulary, and measured by the 40-question benchmark rather than assumed.
