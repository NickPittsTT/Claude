# 02 — Data Model

Conventions:
- `uuid` PKs (`gen_random_uuid()`, or a client-generated id where offline creation is needed)
- `created_at` / `updated_at timestamptz`
- soft delete only where audit requires it
- all money is `numeric(12,2)` plus a `currency char(3)`
- all instants are `timestamptz` plus an IANA `timezone` text on the owning entity
- `country_code char(2)` on events, organisations and regions (expansion-ready)

## ERD (core)

```mermaid
erDiagram
  auth_users ||--|| profiles : has
  profiles ||--o{ user_roles : has
  profiles ||--o{ organisation_members : "member of"
  organisations ||--o{ organisation_members : has
  organisations ||--o{ events : owns
  events ||--o{ event_members : "co-organisers"
  profiles ||--o{ event_members : ""
  events ||--o{ event_distances : offers
  events ||--o{ event_media : has
  events ||--o{ event_sponsors : has
  regions ||--o{ events : "located in"
  countries ||--o{ regions : has
  profiles ||--o{ rsvps : makes
  events ||--o{ rsvps : receives
  profiles ||--o{ event_follows : follows
  profiles ||--o{ check_ins : makes
  events ||--o{ check_ins : receives
  profiles ||--o{ runs : records
  runs }o--o| check_ins : "linked to"
  runs ||--o| community_results : "shared as"
  events ||--o{ community_results : board
  community_results ||--o{ reports : reported
  profiles ||--o{ blocks : blocks
  profiles ||--o{ push_tokens : has
  profiles ||--|| notification_prefs : has
  profiles ||--o{ organiser_mutes : mutes
  plans ||--o{ plan_features : defines
  features ||--o{ plan_features : ""
  plans ||--o{ entitlement_grants : grants
  profiles ||--o{ entitlement_grants : "participant grants"
  events ||--o{ entitlement_grants : "listing grants"
  organisations ||--o{ payment_intents : pays
  payment_intents ||--o{ payment_proofs : has
  payment_intents ||--o| invoices : invoiced
  events ||--o{ event_stats_daily : aggregates
  clubs ||--o{ club_members : has
  profiles ||--o{ club_members : joins
```

## Tables

### Identity & roles
| Table | Key columns | Notes |
|---|---|---|
| `profiles` | `id` (= auth.users.id), `display_name`, `nickname`, `date_of_birth date NOT NULL`, `is_minor` (generated), `home_region_id`, `weight_kg numeric(5,1) NULL`, `health_consent_at`, `emergency_contact_name/phone NULL` (encrypted via pgsodium/Vault column), `profile_public bool default false`, `passport_public bool default false`, `show_attendance_to_clubs bool default false`, `country_code`, `timezone`, `onboarded_at`, `deleted_at` | CHECK: age ≥ 13 at insert. Trigger: if `is_minor`, force the public flags to false |
| `user_roles` | `user_id`, `role app_role`, `granted_by`, `granted_at` | PK (user_id, role). Only Super Admin can insert `platform_admin` |
| `consents` | `user_id`, `kind` (`location`, `health_data`, `public_sharing`, `marketing_push`, `terms`, `privacy`), `version`, `granted_at`, `revoked_at` | Evidence for DPA / store forms |

### Geography & config
| Table | Notes |
|---|---|
| `countries` | `code`, `name`, `currency`, `default_timezone`, `active`. Seed: TT only active |
| `regions` | `id`, `country_code`, `name`, `kind` (`area`/`corporation`), `parent_id`, `sort` |
| `app_settings` | single-row-per-key JSONB: `checkin_radius_m`, `checkin_window_before_min`, `checkin_window_after_min`, `pace_limit_sec_per_km` (155), `free_alert_filter_limit`, `community_disclaimer_text`, legal text versions… Admin-editable, audited |
| `legal_documents` | `kind`, `version`, `body_md`, `published_at` — served at `/legal/*` |
| `announcements` | platform-wide messages + push |

### Organisers & events
| Table | Key columns / notes |
|---|---|
| `organisations` | `name`, `slug`, `contact_email`, `contact_phone`, `country_code`, `timezone`, `currency`, `status` (`applied`,`verified`,`rejected`,`suspended`), `verified_by`, `rejection_reason` |
| `organisation_members` | `organisation_id`, `user_id`, `role` (`owner`,`manager`) |
| `organiser_applications` | applicant details, ID/document upload refs, 18+ check result, review fields |
| `events` | `organisation_id`, `slug`, `name`, `description`, `starts_at timestamptz`, `timezone`, `country_code`, `region_id`, `venue_name`, `start_location geography(Point)`, `route_polyline NULL`, `registration_url NULL`, `registration_instructions`, `package_collection`, `parking`, `contact_*`, `cause`, `charity_name`, `walk_friendly bool`, `fee_min`, `fee_max`, `currency`, `status` (`draft`,`pending_approval`,`published`,`postponed`,`cancelled`,`unpublished`,`rejected`), `featured_until`, `checkin_radius_m NULL`, `checkin_opens_at`/`closes_at NULL` (overrides), `qr_secret`, `duplicated_from_id`, `published_at`, `approved_by` |
| `event_distances` | `event_id`, `distance_km numeric(5,2)`, `label` ("5K", "10K", "2K fun walk") |
| `event_members` | per-event co-organisers: `event_id`, `user_id`, `role` (`co_organiser`), `invited_by`, `accepted_at` |
| `event_media` | `kind` (`flyer`,`route_map`,`gallery`), `storage_path`, `alt_text` (required — accessibility) |
| `event_sponsors` | `name`, `logo_path`, `url`, `sort` |
| `event_changes` | change history used for "what changed" notifications |
| `race_suggestions` | user-submitted missing races; `status`, `converted_event_id` |
| `duplicate_candidates` | populated by a trigger: same region ± 1 day, trigram name similarity > 0.6 → flagged in the admin queue |

### Participation
| Table | Key columns / notes |
|---|---|
| `rsvps` | `user_id`, `event_id`, `status` (`attending`,`maybe`,`not_attending`), `group_size smallint CHECK 1..20`, `updated_at`. PK (user_id, event_id) |
| `rsvp_history` | append-only, for the organiser trend chart (aggregated nightly to `event_stats_daily`) |
| `event_follows` | implicit when RSVP is attending/maybe; explicit "follow" too |
| `check_ins` | `user_id`, `event_id`, `method` (`gps`,`qr`), `lat/lng rounded to 3 dp`, `accuracy_m`, `checked_in_at`. UNIQUE (user_id, event_id). Insert only via RPC |
| `clubs`, `club_members` | per Question 10 option (a) |
| `event_stats_daily` | `event_id`, `date`, `attending`, `maybe`, `not_attending`, `group_total`, `views`, `shares`, `calendar_adds`, `checkins`, `by_region jsonb` |
| `event_counters` | live counters updated by triggers (fast public "N attending") |

### Runs & community
| Table | Key columns / notes |
|---|---|
| `runs` | `id` (client uuid), `user_id`, `started_at`, `ended_at`, `moving_seconds`, `elapsed_seconds`, `distance_m`, `avg_pace_sec_per_km`, `splits jsonb`, `steps NULL`, `calories_est NULL`, `route_polyline NULL`, `source` (`tracked`,`typed_in`), `event_id NULL`, `check_in_id NULL`, `device_meta jsonb`. Private: RLS owner-only |
| `personal_bests` | materialised per (user, distance bucket 5K/10K/HM/M) — recomputed on run upsert |
| `community_results` | `event_id`, `run_id`, `user_id`, `display_name_snapshot` (nickname for minors), `time_seconds`, `distance_km`, `source` (`tracked`/`typed_in`), `status` (`visible`,`withdrawn`,`removed`), `shared_at`. CHECK via trigger: user has a check-in for the event, pace ≥ the setting, one per user per event |
| `reports` | polymorphic: `target_type` (`community_result`,`event`,`profile`), `target_id`, `reporter_id`, `reason`, `details`, `status`, `resolved_by`, `resolution` |
| `blocks` | `blocker_id`, `blocked_id` — hides the blocked user's entries from the blocker |
| `moderation_actions` | `target_user_id`, `action` (`warn`,`suspend`,`remove_content`), `reason`, `until`, `actor_id` |

### Notifications
| Table | Notes |
|---|---|
| `push_tokens` | `user_id`, `expo_token`, `platform`, `last_seen_at`, `disabled_at` |
| `notification_prefs` | `areas uuid[]`, `distances numeric[]`, `types text[]` (`new_race`,`reminder`,`change`,`announcement`), `quiet_start/quiet_end time`, `timezone`, `reminder_offsets` |
| `organiser_mutes` | `user_id`, `organisation_id` |
| `notification_jobs` | `kind`, `event_id`, `payload`, `status`, `scheduled_for`, `attempts` |
| `notification_deliveries` | `job_id`, `user_id`, `ticket_id`, `receipt_status` (90-day retention) |
| `broadcasts` | organiser-sent messages; `kind` (`promo`,`update`), counted against the plan |

### Plans, entitlements, payments
| Table | Notes |
|---|---|
| `features` | `key` (e.g. `alerts.filter_limit`, `history.full`, `history.yoy`, `card.styles`, `event.broadcasts`, `event.featured`), `value_type` (`bool`,`int`) |
| `plans` | `key` (`participant_free`, `participant_premium`, `organiser_basic`, `organiser_featured`), `audience` (`participant`/`event`), `price`, `currency`, `period` (`month`,`year`,`per_event`), `store_product_ids jsonb`, `active` |
| `plan_features` | `plan_id`, `feature_key`, `value jsonb` — edit to move a feature between free and paid, no code change |
| `entitlement_grants` | `subject_type` (`user`,`event`,`organisation`), `subject_id`, `plan_id`, `source` (`store_apple`,`store_google`,`bank_transfer`,`admin_grant`), `starts_at`, `ends_at`, `external_ref`, `revoked_at` |
| `store_events` | raw RevenueCat webhook payloads (idempotency key = event id) |
| `payment_intents` | `organisation_id`, `event_id NULL`, `plan_id`, `amount`, `currency`, `provider`, `reference` (unique, human-friendly e.g. `RL-7K3Q-92`), `status` (`awaiting_payment`,`proof_submitted`,`paid`,`rejected`,`expired`) |
| `payment_proofs` | `payment_intent_id`, `storage_path`, `uploaded_by`, `note` |
| `invoices` | `number`, `payment_intent_id`, `issued_at`, `pdf_path`, tax fields |

Resolving entitlements: SQL function `has_feature(subject, feature_key)` / `feature_value(...)` = the most generous value across active grants, falling back to the free plan's value. The same logic lives in TS (`resolveEntitlements`) for UI display; the server decides.

### Support & audit
| Table | Notes |
|---|---|
| `support_tickets` | `user_id NULL`, `email`, `kind` (`contact`,`feedback`,`bug`), `body`, `status`, `assigned_to` |
| `support_messages` | thread replies |
| `audit_log` | `actor_id`, `actor_role`, `action`, `table_name`, `row_id`, `before jsonb`, `after jsonb`, `ip_hash`, `at`. Append-only |
| `rate_limits` | `bucket`, `subject`, `window_start`, `count` |

## RLS summary (enforced in Postgres, tested with pgTAP)

| Resource | Participant | Organiser | Platform Admin | Super Admin | anon |
|---|---|---|---|---|---|
| Published events | read | read; write own org's / co-organised | read/write all, approve | all | read |
| Draft/pending events | — | own org only | all | all | — |
| `rsvps` | own row CRUD | **aggregates only** via `event_counters` / views (+ nicknames of attendees who opted in) | aggregates | aggregates + support lookup | public count only |
| `runs` | own only | — | — (no access to private runs) | — | — |
| `community_results` | read visible (minus blocked users); write own via RPC | read for own events | moderate | moderate | read visible |
| `check_ins` | own; insert via RPC | count for own events | read | read | — |
| `profiles` | own; others only if `profile_public` (never minors) | nickname of attendees only | read (no emergency contact) | read | — |
| `payment_*`, `invoices` | — | own org | read/update status | all | — |
| `app_settings`, `plans`, `plan_features` | read public keys | read | update (audited) | all | read public keys |
| `user_roles` | read own | read own | read; cannot grant admin | grant/revoke | — |
| `audit_log` | — | own actions | read | read | — |

## Seed data (Phase 1)

- One account per role: `super@`, `admin@`, `organiser@` (verified org) and `organiser.pending@`, `runner.adult@`, `runner.teen@` (age 15), plus a blocked-under-13 test path with no account.
- ~20 realistic T&T events across areas, using dates relative to "today" so the seed never goes stale:
  - Queen's Park Savannah 5K
  - Chaguaramas Boardwalk 5K
  - San Fernando Hill Run
  - Arima Velodrome Fun Run
  - Pigeon Point (Tobago) Sunrise 5K
  - Couva Charity Walk/Run
  - Point Fortin Borough Day 5K
- Some events are postponed or cancelled, some are past (to exercise check-ins, history and the community board). All are clearly marked as SAMPLE data and the seed runs only outside production. Event names are generic/fictional so they don't imply endorsement by real organisers.
