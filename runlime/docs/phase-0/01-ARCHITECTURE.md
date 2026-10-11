# 01 — Architecture

## 1. Stack (versions checked on npm, 2026-10-11)

| Layer | Choice | Version at time of writing | Why / alternative |
|---|---|---|---|
| Mobile | Expo (managed, CNG) + expo-router | SDK 57 (`expo@57.0.27`) | One codebase, EAS cloud builds (no Mac needed). Alt: bare RN, which means more native maintenance for no gain |
| Language | TypeScript | the version pinned by SDK 57 | TS 7.0 just shipped; wait for ecosystem |
| Web | Next.js App Router | 16.4.x | SSR for public event pages (link previews / SEO), same React skills. Alt: Remix, SvelteKit |
| UI (web) | Tailwind CSS + shadcn/ui (Radix: accessible primitives) | current | Accessible defaults, low cost |
| UI (mobile) | Own small component kit on `react-native` primitives + `react-native-reanimated` only where needed | — | Keeps low-end Android performance predictable |
| Backend | Supabase: Postgres 17, Auth, Storage, Edge Functions (Deno), pg_cron, pg_net | supabase-js 2.117 / CLI 2.120 | RLS, built-in auth providers incl. Apple |
| Data fetching | TanStack Query (both apps), persisted cache on mobile | v5 | Offline calendar |
| Validation | zod schemas in `packages/shared` | v4 | One schema for client forms, edge functions and DB-input checks |
| Local DB (mobile) | expo-sqlite | 57.x | Offline run storage + outbox |
| Maps | react-native-maps (Apple Maps iOS / Google Maps Android) | — | $0 native SDKs. Alt: MapLibre + MapTiler (offline packs possible, paid at scale) |
| Location | expo-location + expo-task-manager | 57.x | See §5 |
| Steps | expo-sensors `Pedometer` | 57.x | iOS CMPedometer / Android step counter |
| Subscriptions | RevenueCat `react-native-purchases` | 10.12 | Wraps StoreKit 2 + Play Billing, webhooks |
| Push | expo-notifications + Expo Push API | 57.x | Free, handles APNs/FCM |
| Errors | Sentry (`@sentry/react-native`, `@sentry/nextjs`) | current | |
| Analytics | PostHog (cookieless on web, no autocapture of text inputs) | current | Privacy-respecting; self-host option later |
| Email | Resend (SMTP for Supabase Auth + transactional) | — | |
| Hosting (web) | Vercel Pro | — | Hobby forbids commercial use |
| CI | GitHub Actions + EAS Build/Submit | — | |

## 2. Repository layout

```
runlime/
├─ apps/
│  ├─ mobile/            Expo app (participants; organisers see a read-only "use the portal" note)
│  └─ web/               Next.js: / (landing), /e/[slug] (public event), /legal/*, /support,
│                         /delete-account, /organiser/*, /admin/*
├─ packages/
│  ├─ shared/            db.types.ts (generated), zod schemas, rules/ (pure functions), i18n strings
│  ├─ ui-tokens/         colour/spacing/type tokens consumed by both apps
│  └─ config/            eslint, tsconfig, prettier presets
├─ supabase/
│  ├─ migrations/        SQL, forward-only, numbered
│  ├─ functions/         Edge Functions (Deno): revenuecat-webhook, push-fanout, broadcast,
│  │                      checkin, share-result, export-user-data, delete-account, og-image
│  ├─ tests/             pgTAP RLS tests
│  └─ seed/              seed.sql + seed.ts (realistic T&T events, one account per role)
├─ e2e/
│  ├─ web/               Playwright
│  └─ mobile/            Maestro flows
├─ docs/
├─ CLAUDE.md
├─ BACKLOG.md
└─ README.md
```

### Business rules live in exactly one place
Pure TypeScript functions in `packages/shared/rules/`:

- `ageGate(dob, today)`
- `isWithinCheckinWindow(eventStart, now, settings)`
- `isWithinCheckinRadius(point, start, radiusM)`
- `isPaceAllowed(seconds, km, limit)`
- `resolveEntitlements(grants, plans, now)`
- `rsvpCounts(rsvps)`
- `metCalories(...)`

Each is unit-tested with Vitest. **Server enforcement duplicates the critical ones in SQL** (age, check-in uniqueness, pace limit, entitlement checks) via constraints, triggers or `SECURITY DEFINER` RPCs. The client can never be the only gate. Edge Functions import the same TS functions, so the logic can't drift.

## 3. Environments

| Env | Supabase | Web | Mobile | Data |
|---|---|---|---|---|
| local | `supabase start` (Docker) | `next dev` | Expo dev client | seed |
| staging | Supabase project `runlime-staging` | Vercel preview + `staging.<domain>` | EAS channel `staging`, internal distribution / Play internal track / TestFlight | seed + test accounts |
| production | Supabase project `runlime-prod` (Pro, daily backups, optional PITR) | `<domain>` | EAS channel `production`, store builds | real |

- Secrets live in Supabase Vault / Edge Function secrets, Vercel env vars, EAS secrets and GitHub Actions secrets. None are committed; `.env.example` lists every variable.
- Migrations reach staging when a PR merges, and production through a manual "promote" workflow with an approval gate.

## 4. Auth, roles, RLS

- Supabase Auth providers: email (OTP code/magic link), Google, Apple (+ phone if Question 6 says so).
- **Age gate before account creation.** The DOB is collected on the device. If the user is under 13, sign-up stops and nothing is sent to the server. For 13+, sign-up passes the DOB in user metadata. A `before-user-created` Auth hook (Edge Function) re-validates the age (≥13) and rejects otherwise. A DB constraint on `profiles.date_of_birth` double-checks.
  - Social sign-in: the OAuth flow runs first. Then a mandatory DOB screen appears *before any profile row exists*. Until DOB is confirmed, the account is "pending", has no access through RLS, and is purged after 24h. An under-13 DOB deletes the auth user immediately.
- Roles: `app_role` enum (`super_admin`, `platform_admin`, `organiser`, `participant`) in `user_roles`. A custom access-token hook adds roles to the JWT claims. RLS policies use `auth.jwt()` claims plus helper functions (`is_admin()`, `is_event_member(event_id)`) marked `STABLE SECURITY DEFINER` with a fixed `search_path`.
- Super Admin can't be granted from the UI. It's set once by a seed/CLI script. Only Super Admin can grant or revoke `platform_admin` (enforced in the RPC + RLS).
- Organiser = role + an `organisations` membership + a verified organisation. They must be 18+, checked from DOB when they apply.

## 5. Run tracker & background location (D6)

```
[Start pressed] → disclosure screen (first time only) → requestForegroundPermissionsAsync
   → startLocationUpdatesAsync('run-track', {
        accuracy: BestForNavigation, distanceInterval: 5, timeInterval: 2000,
        foregroundService: { notificationTitle: 'RunLime is tracking your run', ... },  // Android
        showsBackgroundLocationIndicator: true, pausesUpdatesAutomatically: false,       // iOS
        activityType: Fitness })
   → TaskManager task writes points to SQLite (batched) — works with screen off, no network
[Pause] stop distance accumulation, keep task alive  [Stop] stop task, finalise run row in SQLite
[Sync worker] uploads finished runs via RPC upsert (client UUID = idempotent), retries with backoff
```

- Distance: haversine over Kalman-smoothed points. Points with accuracy worse than 25 m are dropped, and so are speed spikes over 12 m/s.
- Splits per km, moving time vs elapsed time, pace.
- Route storage: Google encoded polyline (~5–10 KB per 5K run) in `runs.route_polyline` + a simplified version for thumbnails. Raw points are kept on the device only.
- Steps: `Pedometer.watchStepCount` during the run (an estimate).
- Calories: MET formula, shown only if the user entered weight and consented to health data.

## 6. Notifications design

- `notification_jobs` queue table, filled by triggers or RPCs:
  - event published → "new race" broadcast job
  - event updated (time/venue/status) → "change" job to followers
  - pg_cron each 15 min → reminder jobs (7d, 1d, morning of)
- Edge Function `push-fanout` (pg_cron every minute):
  - resolves the audience with SQL that respects preferences (areas, distances, type), quiet hours (in the user's timezone), organiser mutes, entitlement-based alert limits, and per-user daily caps
  - sends in batches of 100 to the Expo Push API
  - stores receipts and prunes dead tokens
- Broadcast limits: `broadcast_quota_used` per event vs the plan feature `broadcasts_per_event`. Changes/cancellations are always allowed (`kind = 'update'` bypasses the quota but is rate-limited to 1 per 10 min per event, which stops spam disguised as updates).

## 7. Check-in

- RPC `check_in(event_id, method, lat?, lng?, qr_token?)`, `SECURITY DEFINER`:
  - validates the time window and radius from `app_settings` / event overrides
  - for QR, the token = HMAC(event_id, rotating secret), printed by the organiser
  - unique `(user_id, event_id)` constraint
  - GPS spoofing can't be fully prevented. We record accuracy and method so we can audit.
- QR codes encode `https://<domain>/c/<event_slug>?t=<token>`. Scanning with the phone camera opens the app via universal/app links; in-app scanning uses `expo-camera`.

## 8. Public web & deep links

- `/e/[slug]` is server-rendered with Open Graph tags and an OG image generated by `next/og`.
- Smart banner + "Open in app" button. iOS universal links (`apple-app-site-association`) and Android App Links (`assetlinks.json`) are served from the web app.
- Unpublished/cancelled events: cancelled events stay visible with a banner. Unpublished ones return 404.

## 9. Payments layer (organiser)

- `payment_intents` (generic: `provider = 'bank_transfer' | 'card_gateway_x'`, amount, currency, reference, status)
- `payment_proofs` (uploads, private bucket)
- `invoices` (number sequence per country)
- An admin marking a payment Paid → trigger creates the `entitlement_grants` for the event/organisation with `starts_at` / `ends_at`. Expiry is evaluated at read time, and a nightly job flips the status for reporting.
- Adding a card gateway later = a new `provider` value + one webhook Edge Function. The schema doesn't change.

## 10. Security & privacy controls

- RLS on every table; `anon` can only read published events, public organiser profiles, and public aggregates via views.
- Rate limiting:
  - Supabase Auth built-in limits
  - per-user RPC limits through a `rate_limits` table + a `check_rate_limit()` helper
  - Vercel middleware for public forms (suggest-a-race, contact)
  - Cloudflare Turnstile on public web forms
- Storage buckets:
  - `public-media` (flyers, route images, sponsor logos — published events only; uploads checked for MIME and size, ≤5 MB, re-encoded server-side)
  - `private-proofs` (payment proofs: organiser + admins only)
  - `exports` (signed URLs, auto-deleted after 7 days)
- Minors (13–17):
  - computed `is_minor` (generated from DOB)
  - privacy toggles locked to private
  - community entries forced to nickname
  - never included in "friends/clubs attending" lists shown to non-members
  - location rounded in any shared output
- Audit log: a generic trigger on admin/organiser-writable tables writes `audit_log(actor, action, table, row_id, before, after, at)`. The table is append-only (no UPDATE/DELETE grants for anyone, Super Admin included).
- Account deletion:
  - the user request fires an Edge Function that deletes the auth user, which cascades to personal rows
  - aggregate counts are kept anonymised (RSVP counts are decremented; community entries are removed)
  - payment records are retained for 7 years for tax, with the person detached
- Data export: an Edge Function builds a JSON + GPX zip and emails a signed link.
