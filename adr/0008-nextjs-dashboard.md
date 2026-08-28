# ADR-0008 · Next.js dashboard, not Streamlit

**Status:** Accepted · Phase 11

## Context
A solo backend-weighted build is tempted toward Streamlit or Gradio. The dashboard is, however, the only part of the system a judge experiences directly, and the same app must also serve the public customer-facing recovery page on a phone.

## Decision
Next.js 15 (App Router) + TypeScript + Tailwind + shadcn/ui, deployed on Vercel. One app serves both the merchant dashboard and the public recovery microsite.

## Consequences
- **Positive:** looks like software rather than a notebook; server-sent events give a genuinely live demo; the recovery page is mobile-first and shareable; free hosting with a public URL.
- **Negative:** a real frontend costs roughly a day. Budgeted, and bounded by using a component library rather than bespoke design.
