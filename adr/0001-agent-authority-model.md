# ADR-0001 · Deterministic core, LLM at the edges

**Status:** Accepted · Phase 00

## Context
Recoup takes actions that cost money and touch customers. The default hackathon architecture is an LLM agent with tools, where the model decides what to call. That architecture is fast to build and impossible to prove correct: the same input can produce different actions, and "why did it do that" has no answer beyond a transcript.

## Decision
The LLM may **classify-assist, draft, extract, converse within a bounded graph, and explain from retrieved records**. It may never decide whether an action is permitted, choose whether to spend money, or execute anything. Every state transition is deterministic, unit-tested Python behind the policy engine.

An architecture test enforces this: no module under `llm/` may import the execution dispatcher, and no execution path exists without a logged policy verdict.

## Consequences
- **Positive:** every action is reproducible, explainable, and testable; model outage degrades polish, never safety; the authority boundary is a table in the FRD that a reviewer can check.
- **Negative:** less "agentic" flexibility; new intervention types require code plus policy, not a prompt change. Accepted deliberately — that friction is the point when money is involved.
