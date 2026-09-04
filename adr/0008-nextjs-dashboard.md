# ADR-0008 · Next.js dashboard, not Streamlit

**Status:** Accepted · Phase 11

## Context
A solo backend-weighted build is tempted toward Streamlit or Gradio. The dashboard is, however, the only part of the system a judge experiences directly, and the same app must also serve the public customer-facing recovery page on a phone.

## Decision
Next.js 15 (App Router) + TypeScript + Tailwind + shadcn/ui, deployed on Vercel. One app serves both the merchant dashboard and the public recovery microsite.

## Consequences
- **Positive:** looks like software rather than a notebook; server-sent events give a genuinely live demo; the recovery page is mobile-first and shareable; free hosting with a public URL.
- **Negative:** a real frontend costs roughly a day. Budgeted, and bounded by using a component library rather than bespoke design.

## Update (Phase 12)
Two parts of the original decision didn't ship as planned, both worth naming rather than
quietly leaving the ADR wrong: the shadcn/ui component library was dropped in favor of a
small hand-rolled `web/components/ui.tsx` (no `components.json`, no Radix dependency —
the "bounded by a component library" premise above didn't hold up against how specific the
dashboard's own design system needed to be), and live Vercel deployment was cut on the
final day in favor of hardening the local path (see `docs/09-INCIDENT-LOG.md`'s standing
reflection on that trade-off). The framework choice itself (Next.js, now on 15→16) and the
one-app-serves-both-surfaces call both held.
