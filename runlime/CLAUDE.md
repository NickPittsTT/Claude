# CLAUDE.md — RunLime

Living record of conventions and decisions. Update at the end of every phase.

## Product guardrails (from the master prompt; do not drift)
- Market: Trinidad & Tobago only at launch; tz America/Port_of_Spain; TTD; km.
- We do NOT handle registration, entry fees, or official timing. No timing-company integrations.
- Roles: super_admin, platform_admin, organiser (18+), participant (13+). Enforce in Postgres RLS, not only in UI.
- Under-13: block at sign-up and store nothing.
- 13–17: private by default, nickname only, no precise location shown to others.
- Participant Premium sold ONLY via Apple IAP / Google Play Billing (RevenueCat).
- Organiser payments ONLY via web portal bank transfer. The mobile app has no payment links or wording for them.
- Community times: opt-in, check-in required, Tracked/Typed in label, pace limit (setting), withdrawable, report/block, no prizes, disclaimer always shown.
- All limits, prices, radii, windows, and legal text live in DB settings, never hardcoded.
- New ideas go to BACKLOG.md, not into the build.
- Work in phases; stop for owner approval after each phase.

## Process
- Before reporting a phase complete: run tests, then state exactly what was and was not tested.
- Flag every assumption. Offer options with trade-offs when the decision is the owner's.
- No placeholder screens or mock data in production code. Seed data runs only outside production.
- Recheck Apple/Google policy before each relevant phase.

## Decisions log
| Date | Decision | Status |
|---|---|---|
| 2026-10-11 | Phase 0 docs in `docs/phase-0/`. Stack: Expo SDK 57, Next.js 16, Supabase, pnpm + Turborepo | Proposed |
| 2026-10-11 | Run tracking uses Android foreground service + iOS When-In-Use background mode; no "Always" permission (D6) | Proposed — awaiting owner |
| 2026-10-11 | Business rules as pure TS in `packages/shared/rules`, critical ones re-enforced in SQL | Proposed |
| 2026-10-11 | Native maps (Apple/Google) at $0; no offline tiles in v1 | Proposed — awaiting owner |

## Conventions (to be filled from Phase 1)
- Code: TypeScript strict; zod at every boundary; no `any`.
- DB: snake_case; forward-only migrations; every table has RLS enabled plus a pgTAP test.
- Time: store `timestamptz` UTC plus an IANA tz; format in the event's tz.
- Money: `numeric(12,2)` plus currency.
- Commits: conventional commits.
