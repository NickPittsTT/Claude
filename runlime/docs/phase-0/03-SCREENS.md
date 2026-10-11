# 03 — Screen List

Phase tag = the phase in which the screen is built.

## Mobile app (participants)

### Auth & onboarding
| # | Screen | Phase |
|---|---|---|
| M1 | Splash / boot (loads cached calendar) | 1 |
| M2 | Welcome (sign in with Apple / Google / Email [/ Phone]) | 1 |
| M3 | Email code entry | 1 |
| M4 | Date of birth (neutral age gate: plain date picker, no hint of the threshold) | 1 |
| M5 | "Sorry, you can't create an account" (under-13; no data stored) | 1 |
| M6 | Profile basics (display name, nickname, home area) | 1 |
| M7 | Terms & Privacy acceptance | 1 |
| M8 | Walkthrough (3 cards) | 2 |
| M9 | Pick areas & distances of interest | 2 |
| M10 | Notification opt-in explainer → system prompt | 3 |

### Main tabs: Calendar · My Races · Run · History · Profile
| # | Screen | Phase |
|---|---|---|
| M11 | Calendar: list view (sticky month headers, "N attending") | 2 |
| M12 | Calendar: month view | 2 |
| M13 | Filters sheet (date, area, distance, price, cause, walk-friendly) | 2 |
| M14 | Search | 2 |
| M15 | Near me (map with event pins + list) | 2 |
| M16 | Event detail (all fields from spec D, map, route, sponsors, status banner, add to calendar, share) | 2 |
| M17 | Route map full-screen | 2 |
| M18 | RSVP sheet (Attending/Maybe/Not + group size stepper) | 3 |
| M19 | Who's going (count + opted-in club members) | 3 |
| M20 | My Races (upcoming RSVPs, past check-ins) | 3 |
| M21 | Notification inbox | 3 |
| M22 | Check-in (GPS check / "Scan QR" button / result state) | 4 |
| M23 | QR scanner | 4 |
| M24 | Location disclosure (prominent, before the OS prompt; battery note) | 5 |
| M25 | Run: ready (GPS signal indicator, link to today's checked-in event) | 5 |
| M26 | Run: live (time, distance, pace, map, pause/stop; lock-screen-friendly) | 5 |
| M27 | Run: summary (map, splits, steps est., calories est., save/discard, link to event) | 5 |
| M28 | History list + totals header (km, steps, calories, time) | 5 |
| M29 | Run detail | 5 |
| M30 | Personal bests & pace trend | 5 |
| M31 | Same-race year-on-year | 5 |
| M32 | Add manual time ("Typed in") | 5 |
| M33 | Share to community board (consent, nickname choice, disclaimer) | 6 |
| M34 | Community board per event (disclaimer always shown; Tracked/Typed-in labels; report/block) | 6 |
| M35 | Report sheet | 6 |
| M36 | Race passport | 6 |
| M37 | Finisher card builder + share | 6 |
| M38 | Premium paywall (store prices, restore, manage subscription, terms links) | 7 |

### Profile & settings
| # | Screen | Phase |
|---|---|---|
| M39 | Profile view/edit | 1 |
| M40 | Privacy settings (profile, passport, show attendance to clubs; locked for minors) | 3 |
| M41 | Notification preferences (areas, distances, types, quiet hours, muted organisers) | 3 |
| M42 | Health data (weight, consent, revoke) | 5 |
| M43 | Emergency contact | 1 |
| M44 | Blocked users | 6 |
| M45 | Clubs (browse/join/leave) | 3 |
| M46 | Data export request | 1 (Edge Function) / 8 (polish) |
| M47 | Delete account (two-step confirm) | 1 |
| M48 | Help: FAQ, contact support, send feedback, suggest a race | 7 (suggest a race: 2) |
| M49 | About / legal / health disclaimer / licences | 8 |
| M50 | Offline banner + sync status component (global) | 2/5 |

## Organiser portal (web, responsive) — `/organiser`
| # | Screen | Phase |
|---|---|---|
| O1 | Apply to be an organiser (org details, 18+ confirmation, documents) | 1 |
| O2 | Application status | 1 |
| O3 | Dashboard (my events by status, alerts, plan status) | 2 |
| O4 | Event editor: multi-step (basics → location & route → details → media & sponsors → review/submit) | 2 |
| O5 | Route drawer (Google Maps JS) or route image upload | 2 |
| O6 | Duplicate event (annual) | 2 |
| O7 | Postpone / cancel flow (requires reason; triggers follower notification) | 2/3 |
| O8 | Co-organisers (invite by email, accept, remove) | 2 |
| O9 | Broadcasts (compose, quota remaining, history) | 3 |
| O10 | Attendance dashboard (counts, trend chart, by area, views, shares, calendar adds, check-ins, conversion, CSV export) | 3 |
| O11 | Check-in QR (print-ready A4 poster) | 4 |
| O12 | Community board (read-only, report) | 6 |
| O13 | Billing: choose plan → reference & bank details → upload proof → status | 7 |
| O14 | Invoices | 7 |
| O15 | Organisation settings & members | 2 |

## Admin portal (web) — `/admin`
| # | Screen | Phase |
|---|---|---|
| A1 | Dashboard (users, actives, events by status, RSVPs, check-ins, runs, revenue, pending queues) | 3 (basic) / 7 (revenue) |
| A2 | Organiser verification queue (approve / reject with reason / suspend) | 1 |
| A3 | Event approval queue (+ duplicate warnings; edit, publish, reject, unpublish, feature) | 2 |
| A4 | Create event on behalf of organiser | 2 |
| A5 | Race suggestions queue (convert to event) | 2 |
| A6 | Users (search, view, suspend; grant/revoke Platform Admin: Super Admin only) | 1 |
| A7 | Moderation queue (reports → remove/warn/suspend) | 6 |
| A8 | Organiser payments (proof viewer, mark Paid/Rejected) | 7 |
| A9 | Plans & features matrix editor | 7 |
| A10 | Settings (check-in, pace, limits, regions, disclaimers, legal docs) | 2–7 incrementally |
| A11 | Announcements | 3 |
| A12 | Support inbox | 7 |
| A13 | Audit log (filterable, read-only, before/after diff) | 7 |

## Public web
| # | Page | Phase |
|---|---|---|
| W1 | Landing (store badges, upcoming featured races) | 2 |
| W2 | Public event page `/e/[slug]` (OG image, open-in-app) | 2 |
| W3 | Public calendar `/races` (read-only, SEO) | 2 |
| W4 | Privacy Policy, Terms, Moderation Policy, Health Disclaimer, Community Times notice | 1 (draft) / 8 (final) |
| W5 | Support / contact / FAQ | 7 |
| W6 | Account deletion request (Google requirement) | 1 |
| W7 | QR landing `/c/[slug]` (opens app or explains how to install) | 4 |
