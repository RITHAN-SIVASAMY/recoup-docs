# ADR-0009 · Groq as grounded Q&A's drafter

**Status:** Accepted · Phase 12

## Context
Every other LLM path (message drafting, PTP extraction, voice) stays on Anthropic. Grounded Q&A (`audit/qa.py`'s `ask()`) was already built so the model call is a swappable `Drafter` callable — `ask()` itself owns citation validation and refusal, never trusting the drafter's own claims (ADR-0001's authority boundary: the LLM answers "why did this happen?", code enforces citations and refusal). That separation means a provider swap on this one path touches nothing else in the authority boundary and needs no change to the enforcement logic or its tests.

## Decision
`llm/qa.py`'s `answer_grounded_question()` calls Groq's OpenAI-compatible chat completions endpoint (`llama-3.3-70b-versatile` by default) instead of Anthropic. Same prompt, same redaction, same JSON-schema validation, same retry-once-then-degrade-to-`None` contract as before. Gated on `GROQ_API_KEY`; `tests/llm_eval/test_grounded_qa_golden.py` gates on the same key instead of `ANTHROPIC_API_KEY`.

## Consequences
- **Positive:** materially lower per-question latency for the Insights tab's live Q&A widget, which matters for a judge clicking around during the demo; zero blast radius on the authority-boundary tests, since the citation/refusal contract lives entirely in `audit/qa.py`, not in the drafter.
- **Negative:** a second third-party LLM dependency to keep configured and within rate limits; `GroundedAnswer`'s JSON-mode reliability is now Groq/Llama's, not Claude's, so a citation-format regression is possible if the golden set isn't re-run after a model version change.
