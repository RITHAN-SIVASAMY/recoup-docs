# Architecture Decision Records

One file per irreversible decision. Format: context → decision → consequences → status.

| ADR | Decision | Status |
|---|---|---|
| [0001](0001-agent-authority-model.md) | Deterministic core, LLM at the edges | Accepted |
| [0002](0002-event-sourcing.md) | Event sourcing with a hash chain | Accepted |
| [0003](0003-no-temporal.md) | Postgres + Redis + arq instead of Temporal | Accepted |
| [0004](0004-policy-as-code.md) | Versioned YAML policy-as-code | Accepted |
| [0005](0005-no-vector-db.md) | Postgres FTS for grounded retrieval | Accepted |
| [0006](0006-simulator-first-channels.md) | Simulator-first channel adapters | Accepted |
| [0007](0007-uplift-x-learner.md) | X-learner for uplift estimation | Accepted |
| [0008](0008-nextjs-dashboard.md) | Next.js dashboard, not Streamlit | Accepted |
