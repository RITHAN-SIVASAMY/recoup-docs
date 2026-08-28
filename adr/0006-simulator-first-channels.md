# ADR-0006 · Simulator-first channel adapters

**Status:** Accepted · Phase 06

## Context
Recovery requires SMS, WhatsApp, email and voice. Live providers cost money, need approvals, rate-limit unpredictably, and fail on demo day.

## Decision
Define a `ChannelPort` protocol. The default implementation is a **seeded deterministic simulator** with per-segment response curves. Live adapters (Twilio, WhatsApp Cloud API, Resend) implement the same port and are enabled by a flag.

## Consequences
- **Positive:** the whole system is testable and demonstrable offline, at zero cost, reproducibly; the demo never depends on a third party; response curves make measurement experiments possible at all.
- **Negative:** simulated response behaviour is an assumption, and measured lift partly reflects it. Stated as a threat to validity in the evaluation protocol rather than hidden.
