# ADR-0004 · Versioned YAML policy-as-code

**Status:** Accepted · Phase 04

## Context
Escalation ladders, stopping rules and regulatory guardrails could live as Python conditionals, as rows in a database, or as prompt instructions. Prompt-resident rules are unverifiable. Scattered conditionals are unreviewable by the compliance owner who actually needs to read them.

## Decision
Rules live in versioned YAML under `policies/`, content-hashed into `policy_version`, evaluated by a pure function that returns ALLOW / DENY / REQUIRE_APPROVAL plus the rule ID. Regulatory rules live in a **separate file** from merchant-tunable business policy.

## Consequences
- **Positive:** diffable in git; readable by a non-engineer; testable with property-based tests; a historical decision is explainable with the exact rules in force at the time; the what-if simulator replays history through a candidate policy.
- **Negative:** an expressiveness ceiling. Any rule that cannot be expressed in YAML is escalated to a human decision and an ADR, never hardcoded quietly.
