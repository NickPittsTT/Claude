# RunLime — Phase 0: Discovery & Design

Status: **Awaiting owner review. No application code has been written.**
Date: 2026-10-11

| Doc | Contents |
|---|---|
| `00-OVERVIEW.md` (this file) | Questions for you, decisions needed, assumptions, policy findings |
| `01-ARCHITECTURE.md` | Stack (verified versions), repo layout, environments, key technical designs |
| `02-DATA-MODEL.md` | ERD, every table, RLS strategy, business-rule placement |
| `03-SCREENS.md` | Full screen list for mobile, organiser portal, admin portal, public web |
| `04-BRAND.md` | Three colour + logo directions with measured contrast ratios |
| `05-COSTS.md` | Running cost per month at 1k / 10k / 50k users |
| `../../CLAUDE.md` | Conventions & decisions log (starts now, updated each phase) |
| `../../BACKLOG.md` | Ideas outside scope |

---

## 1. Questions I need answered before Phase 1

Grouped by urgency. **Bold = blocks Phase 1.**

### Business & legal
1. **Repository.** This repo (`nickpittstt/claude`) currently holds an unrelated game (`monster_kitchen.html`). I put RunLime in a `runlime/` subfolder for now. Do you want a dedicated repo (recommended: `runlime`)? Moving later is cheap now and costly after CI is set up.
2. **Legal entity.** Will you publish as an individual or a registered company?
   - The Google Play **12-tester / 14-day closed test** rule applies only to *personal* accounts. Organisation accounts are exempt, but they need a D-U-N-S number (free, takes about 1–2 weeks).
   - On Apple, an individual account shows your personal name as the seller.
   - Recommendation: register a company/business name and get D-U-N-S **now**. It's the longest lead-time item.
3. **Bank details for organiser payments.** Which bank and account name? Do you need to issue **VAT** invoices (T&T VAT registration threshold is TT$600k turnover)? This decides the invoice template.
4. **Support email / domain.** Do you own a domain yet? Store listings, Privacy Policy URLs, auth emails and deep links all need one before Phase 2.
5. **Who drafts the legal text?** I will draft the Privacy Policy, Terms, Moderation Policy and Health Disclaimer from the real data model. **They must be reviewed by a T&T attorney.** Note: the Data Protection Act 2011 is only *partially* proclaimed. I'll follow its General Privacy Principles as you asked, but the attorney should confirm current obligations.

### Product
6. **Sign-in by phone.** Twilio's published rate to T&T is **about US$0.30 per SMS**. At 50k users, phone OTP alone could cost US$500–900/month (see `05-COSTS.md`). Options:
   - (a) **Recommended:** launch with Email (magic link or code) + Google + Apple. Phone goes to the backlog.
   - (b) Phone via WhatsApp OTP (cheaper per message, extra setup with Meta).
   - (c) Phone SMS as specified, accepting the cost.
7. **Premium: what's in it at launch?** The entitlement system makes this configurable, but I need a starting set for store listings and review notes. Proposal:
   - Free: calendar, RSVP, check-in, run tracking, 3 area/distance alert filters, history for the last 12 months.
   - Premium: unlimited alert filters, full history + year-on-year comparison, pace trend charts, finisher card styles, race passport themes.
   - Price suggestion: US$2.99/month or US$19.99/year (store price tiers are set in USD/local tiers, not freely in TTD).
8. **Organiser listing prices (TTD).** Starting values for Basic and Featured per event? (They're configurable, but I need seed values.) Is there a free tier for charity races?
9. **Event approval SLA.** Should events from verified organisers ever auto-publish, or always go through admin approval? The spec says always. I'll build "always", with a settings toggle to change it later.
10. **"Friends" model.** Spec E says "show which friends or clubs are attending (opt-in)". There's no friends/follow feature elsewhere in the spec. Options:
    - (a) **Recommended for v1:** *Clubs only*. Users join a public running club (admin-created list) and opt in to show attendance to club members.
    - (b) Mutual friend requests: a bigger feature, and more moderation surface for 13–17 users.
11. **Areas/regions list.** Should I seed the 14 regional corporations + 5 boroughs of Trinidad + Tobago (THA)? Or friendlier running areas ("North-West", "East-West Corridor", "Central", "South", "Tobago")? Recommendation: friendly areas for filters, with the corporation stored for analytics.
12. **Community board for 13–17s.** The spec allows nicknames. Should minors be allowed to share to the community board at all? Recommendation: yes, nickname only, and with a parental-style confirmation screen.

### Technical
13. **Map approach.** See §3 and `01-ARCHITECTURE.md §5`. I recommend native maps (Apple Maps on iOS, Google Maps on Android) at **$0** cost. The trade-off: map tiles don't work offline. The run itself still records offline and the route draws once tiles are cached or connectivity returns. OK?
14. **Background location approach.** See §3. I'm proposing a change that reduces store-review risk. Do you approve?
15. **Hosting for the web portal.** Vercel Hobby **forbids commercial use**, so it's Vercel Pro (US$20/mo) or Cloudflare (cheaper, more setup). Recommendation: Vercel Pro.

---

## 2. Decisions I recommend (approve or change)

| # | Decision | Recommendation | Confidence |
|---|---|---|---|
| D1 | Mobile framework | Expo SDK **57** (current `latest`; SDK 58 is still `next`/beta), React Native 0.8x, expo-router, TypeScript | 0.9 |
| D2 | Web | Next.js **16** (App Router), one app serving the landing page, public event pages, legal pages, organiser portal and admin portal | 0.9 |
| D3 | Backend | Supabase (Postgres 17, Auth, Storage, Edge Functions), RLS on every table | 0.9 |
| D4 | Monorepo | pnpm workspaces + Turborepo; `packages/shared` holds generated DB types, zod schemas and **pure business-rule functions** used by mobile, web and edge functions | 0.85 |
| D5 | Maps | `react-native-maps` (Apple/Google native, $0); Google Maps Embed (free) on public web; Google Maps JS (free tier) only in the organiser route-drawer | 0.75 |
| D6 | Run tracking | `expo-location` + `expo-task-manager` with **Android foreground service + iOS "When In Use" background mode**; no "Always"/`ACCESS_BACKGROUND_LOCATION` | 0.8 |
| D7 | Subscriptions | RevenueCat (`react-native-purchases`); RevenueCat webhook → Supabase Edge Function → `entitlement_grants` | 0.85 |
| D8 | Push | Expo Push Service; fan-out job in an Edge Function, queue table + pg_cron | 0.85 |
| D9 | Offline | Calendar cached via TanStack Query persisted to MMKV/SQLite; runs stored in **expo-sqlite** first, synced by an idempotent upload (client-generated UUIDs) | 0.85 |
| D10 | Observability | Sentry (errors, both apps) + PostHog (product analytics, EU or US cloud, no session replay, IP anonymised) | 0.8 |
| D11 | Email | Resend as Supabase custom SMTP (Supabase's built-in mailer is rate-limited and not for production) | 0.85 |
| D12 | TypeScript version | Use the version Expo SDK 57 pins (5.x). **Not** TypeScript 7.0, which was just released on npm. Its native compiler is too new for parts of the RN toolchain. Revisit in Phase 8. | 0.7 |
| D13 | Testing | Vitest (shared rules + edge functions), pgTAP (RLS policies, tested in SQL), Jest + RNTL (mobile components), Playwright (web E2E), Maestro (mobile E2E flows; runs on Android in CI, no Mac needed) | 0.8 |
| D14 | CI | GitHub Actions: lint, typecheck, unit, pgTAP against a local Supabase in Docker, Playwright. EAS Build triggered on tags. | 0.85 |

---

## 3. Store-policy findings that affect the design

Checked today against current sources (see the Sources list in the chat summary). Each will be rechecked in the phase that implements it.

1. **Background location: proposed design change (D6). Confidence 0.8.**
   - I read the actual `expo-location@57` native source.
   - **Android:** a user-started *foreground service* with a persistent notification does **not** need `ACCESS_BACKGROUND_LOCATION`. The code comment says so explicitly, and the permission check is skipped when `foregroundService` is set.
   - **iOS:** `startLocationUpdatesAsync` only checks the *foreground* ("When In Use") permission. With `UIBackgroundModes: location`, tracking continues with the screen off and the blue location indicator shows.
   - **Result:** we never ask for "Always"/background permission. That avoids Google's background-location declaration and review, the highest-risk part of review for a run tracker. The user sees a simpler prompt.
   - **What remains:**
     - Google's *foreground service type* declaration (`FOREGROUND_SERVICE_LOCATION`) in Play Console, which may still require a short video. I'll still produce the video script you asked for.
     - The prominent disclosure screen (kept anyway).
     - A battery-impact note (kept).
   - This differs from your spec ("request background only when a run starts"). **Your call.** If a device-level test in Phase 5 shows Android OEMs killing the service (Xiaomi/Samsung battery savers), we fall back to your original design.
2. **Google Play closed testing.** For *personal* accounts created after Nov 2023: 12+ testers opted in for 14 continuous days before production access. Organisation accounts are exempt. → Question 2.
3. **Apple age ratings** are now 4+/9+/13+/16+/18+. Expected rating: **13+**. The app has user-generated content (community times, nicknames) and location features, which the new questionnaire asks about. Apple's **Declared Age Range API** is opt-in. I recommend *not* depending on it in v1 (Android has no equivalent) and keeping our own DOB gate. I'll recheck in Phase 8 whether any launch-country law makes it mandatory.
4. **Sign in with Apple** is required because we offer Google sign-in (Guideline 4.8). It's in scope.
5. **Account deletion in-app** is required by both stores. Google also requires a **web** deletion URL. It's in scope: a web deletion page in the portal.
6. **Anti-steering.** The mobile app must not mention organiser bank-transfer payments. Organisers who log into the mobile app see **no** billing UI, only a neutral line: "Manage your events on the RunLime organiser portal." Even that line needs care under Apple 3.1.3, so I'd show the portal URL *without* any payment wording. Organiser management in mobile is out of scope anyway.
7. **Health data.** We'll declare steps/calories/weight as "Health & Fitness" data in both privacy forms. We won't use HealthKit/Health Connect in v1 (backlog). That avoids the extra HealthKit review requirements.

---

## 4. Assumptions (flagged)

- A1. Users are mostly Android (~75–80% in T&T). I'll optimise and test on a low-end Android profile (2–3 GB RAM, Android 10+) first.
- A2. Minimum OS: Android 8.0 (API 26) and iOS 16, following Expo SDK 57 defaults. I'll confirm in Phase 1.
- A3. English only at launch. All strings go through an i18n layer anyway, which keeps expansion cheap.
- A4. A single organiser "organisation" can have multiple members. The spec's "co-organiser per event" is modelled as event-level roles on top of that.
- A5. "Price" filter = entry fee range in the event's currency. Free events are allowed (fee = 0).
- A6. Times are stored as `timestamptz` (UTC) plus the event's IANA timezone, and displayed in the event's timezone.
- A7. Calories estimate: `kcal = MET(pace) × weight_kg × hours`, using the Compendium of Physical Activities running METs. With no weight, we show no calories (no default weight). Labelled "Estimate".
- A8. Community-time pace limit compares *average* pace over the whole event distance, not split pace.
- A9. "Number in my group" includes the account holder. The public attending count = sum of group sizes for Attending. Separate "people" vs "accounts" counts go to organisers.
- A10. Under-13 block: we keep **no record at all**. Not even a hashed email, which means a blocked child could retry with a different DOB. That's the industry norm for a neutral age gate. A local device flag deters immediate retry without storing personal data on our side.

---

## 5. What was tested in Phase 0

Nothing executable. Phase 0 is documents only, as instructed. Verified:
- Current package versions via the npm registry (`expo` latest 57.0.27, `next` 16.4.0, `@supabase/supabase-js` 2.117.3, `react-native-purchases` 10.12.2, `expo-location` 57.0.20).
- `expo-location@57.0.20` native source: the background-permission behaviour behind D6.
- Contrast ratios in `04-BRAND.md`, computed with the WCAG 2.x formula.

Not verified (to be rechecked in the phase that uses them): exact current Apple/Google policy wording, Supabase/Vercel/EAS/RevenueCat/Twilio list prices (taken from vendor pages and third-party summaries dated 2026), Google Maps native-SDK $0 pricing.
