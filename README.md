# recoup-docs

Specification and governance documents for **Recoup** — a risk-aware, causally-measured revenue recovery agent for Razorpay merchants.

This repository is consumed as a **git submodule** at `docs/` inside the [`recoup`](../../../recoup) application repository, so that any commit of the code pins the exact revision of the documents it was built against.

| # | Document | What it answers |
|---|---|---|
| 01 | [Functional Requirements](01-FRD.md) | What the system must do, refuse to do, and prove |
| 02 | [Problem & Differentiation](02-PROBLEM-AND-DIFFERENTIATION.md) | Why anyone needs it, and why a generic AI build doesn't close the gap |
| 03 | [Architecture](03-ARCHITECTURE.md) | How it is put together, and which trade-offs were made on purpose |
| 04 | [Execution Plan](04-EXECUTION-PLAN.md) | The 8-day, 13-phase build plan and the full tech stack |
| 05 | [Evaluation Protocol](05-EVALUATION-PROTOCOL.md) | How every reported number is produced, pre-registered |
| 06 | [Compliance Matrix](06-COMPLIANCE-MATRIX.md) | Every rule → config → code → test |
| 07 | [Demo Script](07-DEMO-SCRIPT.md) | The 5-minute pitch video, beat by beat |
| 08 | [Submission Checklist](08-SUBMISSION-CHECKLIST.md) | Everything the buildathon asks for |
| 09 | [Incident Log](09-INCIDENT-LOG.md) | What broke during the build and how it was fixed |
| — | [ADRs](adr/) | Architecture decision records |

`exports/` holds generated `.docx` versions of the two headline documents for anyone who prefers Word.

## Conventions

- Markdown is the source of truth. `.docx` files are generated (`make docx`), never edited by hand.
- Every document carries a version and a change log line.
- A change to a regulatory constant requires an ADR, because it changes the policy content hash and therefore every downstream decision record.
