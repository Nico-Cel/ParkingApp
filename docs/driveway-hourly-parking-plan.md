# Driveway Hourly Parking — Execution Build Plan

## 1) Product requirements (PRD)

### 1.1 Product vision
Create a two-sided parking marketplace where hosts monetize private parking spots (driveways/garages/lots) and drivers discover, reserve, and pay for hourly parking in congested areas with high reliability and safety.

### 1.2 Goals (MVP, 8–12 weeks, 2–3 engineers)
- Launch iOS + Android apps (React Native Expo) and responsive web app.
- Enable host listing lifecycle: create/edit/publish spot with geocoded location, schedule, pricing, photos, rules.
- Enable driver lifecycle: search → book → pay → park → complete → review.
- Prevent double-bookings through transactional booking + locks.
- Support payments (card + Apple Pay + Google Pay), refunds, no-show handling.
- Provide host-driver in-app messaging with masking and abuse controls.
- Provide admin support console for refunds, disputes, moderation, fraud flags.

### 1.3 Success metrics
- Supply: # published listings per city, activation rate (% hosts publishing within 7 days).
- Demand: search-to-book conversion, booking completion rate, repeat drivers (30-day).
- Reliability: double-booking rate (target 0), payment success rate (>97%), mean support resolution time.
- Trust/safety: fraud report rate, verified-host ratio, dispute rate per 1,000 bookings.

### 1.4 Personas
- Driver commuter/event-goer: needs predictable parking near destination.
- Host homeowner/business: wants easy setup, occupancy, timely payouts.
- Admin support agent: resolves issues quickly, safely, auditable actions.

### 1.5 Scope
#### In-scope (MVP)
- Auth: email/password, magic link; OAuth optional.
- Profiles: host + driver profile, vehicle + plate capture.
- Listing creation with geocode/map pin, availability, hourly pricing, photos, rules.
- Search with map/list, filters (price, distance, EV charger, size), real-time availability.
- Booking with start/end, extension, cancellation, no-show states.
- Stripe Connect marketplace payments + payout scheduling.
- In-app messaging with safety controls.
- Two-sided reviews after completed booking.
- Admin console (lookup, booking management, refunds/disputes, moderation, fraud flags).
- Notifications (push + email).
- GDPR/CCPA export/delete workflows + audit logs.

#### Post-MVP
- Subscriptions/corporate plans, insurance add-on, demand pricing ML, geofence auto-start, smart lock integrations.

### 1.6 Key product decisions/tradeoffs
- **Primary stack**: React Native + NestJS/TypeScript + PostgreSQL/PostGIS for fastest cross-platform delivery.
- **Payment intent model**: authorize before session, capture on start or grace threshold; reduces failed captures but increases hold complexity.
- **Availability model**: recurring schedule + explicit blocks/bookings; simpler than full calendar engine while sufficient for MVP.
- **Moderation level**: hybrid rule-based + manual review in MVP; ML deferred post-MVP.

### 1.7 Assumptions + validation
- Assumption: city-level launch (1–2 metros) limits legal complexity initially.
- Assumption: users accept approximate pin with privacy radius for host addresses until booking confirmation.
- Validate with: concierge pilot (50 hosts/500 drivers), legal review by launch city, weekly funnel audits.

## 2) Non-functional requirements
- Availability: API 99.9% monthly; booking subsystem 99.95%.
- Performance:
  - P95 search API < 400 ms (cached filters + geospatial index).
  - P95 booking create < 600 ms.
  - Mobile cold start < 3 s on median devices.
- Scalability: design for 10k MAU MVP, burst 50 RPS search, 10 RPS bookings.
- Data consistency: strong consistency for booking/payment state transitions.
- Security: OWASP ASVS L2 baseline, encrypted at rest/in transit.
- Privacy: GDPR/CCPA compliant data rights flows.
- Auditability: immutable admin/audit events with actor, timestamp, before/after.
- Accessibility: WCAG 2.1 AA for web admin and key web surfaces; mobile platform accessibility labels.

## 3) Roles, permissions, and key user journeys

### 3.1 Roles
- Guest: browse map/list, no booking.
- Driver: manage profile/vehicles, book/pay/message/review/report.
- Host: manage listings, availability, pricing, messaging, payouts, review drivers.
- Admin Support: read/write across users/bookings/refunds/disputes/moderation.
- Risk Admin (elevated): fraud rules, payout holds, account suspension.

### 3.2 Permission highlights
- Only booking participants can message each other.
- Reviews allowed only after booking completion window and once per side.
- Admin destructive operations require reason code + audit entry.
- Payout schedule edits limited to host + risk admin.

### 3.3 Core journeys
1. Driver onboarding → add vehicle + plate → search spot near destination.
2. Driver selects listing → checks rules/photos/reviews → books + preauth payment.
3. Host receives booking notification → can accept (if manual mode) or auto-confirm.
4. Parking session starts, optional extension prompts before end.
5. Session ends → capture final amount (incl. extension/overstay) → receipts.
6. Mutual reviews open after completion; anti-retaliation publish rules.
7. Support flow for blocked spot/host cancel/no-show/dispute.

## 4) Data model (entities + relationships)

### 4.1 Core entities
- `users` (id, email, password_hash, role_flags, status, kyc_status)
- `profiles` (user_id, name, phone_masked, avatar_url)
- `vehicles` (id, driver_user_id, plate_encrypted, make/model/color/size_class)
- `listings` (id, host_user_id, title, description, address, geom, pin_accuracy_m, is_active)
- `listing_photos` (id, listing_id, object_key, moderation_status)
- `listing_rules` (listing_id, no_oversize, ev_allowed, quiet_hours, cancellation_policy)
- `availability_rules` (id, listing_id, day_of_week, start_local_time, end_local_time, tz)
- `availability_exceptions` (id, listing_id, start_ts, end_ts, type:block/open)
- `pricing_rules` (id, listing_id, base_hourly_cents, peak_windows_json, min_duration_mins, max_duration_mins)
- `bookings` (id, listing_id, driver_user_id, vehicle_id, start_ts, end_ts, status, source)
- `booking_extensions` (id, booking_id, old_end_ts, new_end_ts, reason)
- `parking_events` (id, booking_id, type:arrived/started/ended/no_show/overstay, evidence_photo_key)
- `payments` (id, booking_id, stripe_payment_intent_id, auth_amount_cents, captured_amount_cents, status)
- `refunds` (id, payment_id, amount_cents, reason_code, initiator)
- `payouts` (id, host_user_id, stripe_transfer_id, amount_cents, status, hold_reason)
- `messages` (id, conversation_id, sender_user_id, body_filtered, contains_contact_attempt)
- `conversations` (id, booking_id)
- `reviews` (id, booking_id, reviewer_user_id, reviewee_user_id, rating, tags, body, state)
- `reports` (id, reporter_user_id, target_type, target_id, reason, status)
- `disputes` (id, booking_id, opened_by, category, status, resolution)
- `audit_logs` (id, actor_user_id, action, entity_type, entity_id, before_json, after_json)
- `data_requests` (id, user_id, type:export/delete, status, completed_at)

### 4.2 Relationship summary
- User 1:N Listings, Vehicles, Bookings(as driver), Reviews(given/received), Reports.
- Listing 1:N Photos, AvailabilityRules, Exceptions, PricingRules, Bookings.
- Booking 1:1 Payment, 1:N Extensions, 1:1 Conversation, 0..2 Reviews.
- Admin actions create AuditLogs across all mutable entities.

## 5) API spec (endpoints + request/response examples)

### 5.1 Auth
- `POST /v1/auth/register`
- `POST /v1/auth/login`
- `POST /v1/auth/magic-link/request`
- `POST /v1/auth/token/refresh`

Example request:
```json
{ "email": "driver@example.com", "password": "StrongPass!123" }
```
Example response:
```json
{ "accessToken": "jwt", "refreshToken": "jwt", "user": { "id": "usr_1", "roles": ["driver"] } }
```

### 5.2 Listings
- `POST /v1/hosts/listings`
- `PATCH /v1/hosts/listings/:id`
- `POST /v1/hosts/listings/:id/photos/presign`
- `POST /v1/hosts/listings/:id/publish`
- `GET /v1/listings/:id`

Create listing request:
```json
{
  "title": "Driveway near Civic Center",
  "address": "123 Main St, San Francisco, CA",
  "location": { "lat": 37.78, "lng": -122.41 },
  "baseHourlyCents": 800,
  "rules": { "evAllowed": true, "vehicleSizeMax": "SUV" }
}
```

### 5.3 Search
- `GET /v1/search/listings?lat=..&lng=..&start=..&end=..&maxPrice=..&distanceM=..&ev=true&size=SUV`
Response includes distance, estimated total, availability status.

### 5.4 Booking + session
- `POST /v1/bookings` (transaction + payment intent)
- `POST /v1/bookings/:id/confirm` (if required)
- `POST /v1/bookings/:id/extend`
- `POST /v1/bookings/:id/cancel`
- `POST /v1/bookings/:id/check-in`
- `POST /v1/bookings/:id/check-out`

Book request:
```json
{
  "listingId": "lst_1",
  "vehicleId": "veh_1",
  "startTs": "2026-03-10T17:00:00Z",
  "endTs": "2026-03-10T20:00:00Z",
  "paymentMethodId": "pm_123"
}
```
Book response:
```json
{
  "bookingId": "bkg_1",
  "status": "confirmed",
  "payment": { "intentId": "pi_123", "authorizedCents": 2800 },
  "cancellationPolicy": "moderate"
}
```

### 5.5 Messaging
- `GET /v1/bookings/:id/messages`
- `POST /v1/bookings/:id/messages`
Server masks contacts and flags profanity.

### 5.6 Reviews
- `POST /v1/bookings/:id/reviews`
- `GET /v1/users/:id/reviews-summary`

### 5.7 Admin
- `GET /v1/admin/users/:id`
- `GET /v1/admin/bookings/:id`
- `POST /v1/admin/bookings/:id/refund`
- `POST /v1/admin/disputes/:id/resolve`
- `POST /v1/admin/listings/:id/moderate`
- `POST /v1/admin/users/:id/risk-flags`

### 5.8 Webhooks
- `POST /v1/webhooks/stripe`
- `POST /v1/webhooks/fcm-delivery`

## 6) Architecture (frontend, backend, database, infra, third parties)

### 6.1 Frontend
- Mobile: React Native (Expo), TypeScript, React Query, Zustand/session state, Mapbox SDK.
- Web: Next.js responsive app for driver/host lightweight flows; shared API client and design system.
- Admin: Next.js app (separate route/app) with RBAC guards.

### 6.2 Backend
- NestJS (TypeScript), modular monolith for MVP.
- Modules: Auth, Users, Listings, Search, Booking, Payments, Messaging, Reviews, Admin, TrustSafety.
- Validation: Zod schemas at boundary; OpenAPI generation from DTOs.
- Background jobs: BullMQ (Redis) for notifications, payout batching, moderation queues.

### 6.3 Data layer
- PostgreSQL 15 + PostGIS.
- Prisma ORM + raw SQL where needed for spatial queries/locking.
- Redis cache for search facets, sessions/rate limits, idempotency keys.
- S3 for media with signed upload/download URLs.

### 6.4 Infra
- AWS MVP: ECS/Fargate services, RDS Postgres, ElastiCache Redis, S3, CloudFront, SES, SNS (optional), Secrets Manager.
- CI/CD via GitHub Actions.
- Environments: local (Docker Compose), staging, production.

### 6.5 Third parties
- Stripe Connect Express (marketplace payouts, disputes).
- Mapbox (geocoding/routing/maps).
- Firebase Cloud Messaging (push).
- SendGrid or SES for transactional email.

## 7) Security + privacy + abuse prevention plan
- Auth security: bcrypt/argon2 hashes, refresh token rotation, device/session tracking.
- RBAC enforced in API guards; admin endpoints require scoped JWT claims + MFA for admin users.
- Rate limits per IP/user on auth, messaging, booking attempts.
- Input validation/sanitization; media malware scanning (async).
- PII controls: encrypt phone/email/plate at rest; store minimal PII in logs.
- Contact masking in chat and pre-booking views.
- Abuse: report/block workflows, profanity + contact-sharing detection, temporary chat restrictions.
- Fraud flags: velocity checks (many cards, many cancellations, repeated charge failures), duplicate listing detection (address/image hash).
- GDPR/CCPA: export job (JSON + media links), delete/anonymize workflow with legal-hold exceptions.

## 8) Payments, refunds, disputes, and taxes plan
- Stripe Connect model:
  - Platform as merchant of record for MVP (or destination charges by region constraints).
  - Host onboard via Connect Express; payouts daily/weekly with reserve holds for new hosts.
- Payment flow:
  1. Booking create → PaymentIntent auth for estimated amount + buffer.
  2. Session end/extension → incremental auth/capture final amount.
  3. Receipt issued by email/push.
- Cancellation policies (configurable per listing with platform caps):
  - Flexible: full refund up to start time.
  - Moderate: full refund up to 2h before start, then partial.
- Edge handling:
  - Host last-minute cancel: full driver refund + host penalty fee + ranking penalty.
  - Driver no-show: host gets partial payout after grace period.
  - Overstay: auto-extension if available; else penalty charge and dispute option.
  - Spot blocked/invalid: driver evidence upload; provisional refund while investigating.
- Disputes:
  - In-app dispute ticket linked to booking + evidence.
  - Admin SLA tiers (urgent <4h, standard <24h).
  - Chargebacks create automatic payout hold on related host until resolved.
- Taxes:
  - Use Stripe Tax where supported; fallback table by jurisdiction.
  - Store tax lines per booking for reporting.

## 9) Matching/availability + conflict resolution design
- Availability computed from:
  - Recurring schedule (`availability_rules`)
  - Exceptions/blocks
  - Existing bookings (confirmed/active).
- Search algorithm:
  - PostGIS `ST_DWithin` + filter constraints + time-window availability check.
  - Sort by weighted score: distance, price, host rating, reliability.
- Double-booking prevention:
  - Transaction with `SELECT ... FOR UPDATE` on listing/time bucket rows.
  - Exclusion constraint on booking intervals per listing for active statuses.
  - Idempotency key per booking request.
- Conflict resolution:
  - If concurrent attempts, first commit wins; loser gets 409 with alternatives.
  - Extension conflicts: propose nearby alternatives and grace buffer before penalties.
- GPS mismatch:
  - Arrival check allows tolerance radius; driver can submit "pin incorrect" issue, auto-create support case.

## 10) Ratings/reviews design (two-sided) + anti-gaming rules
- Two reviews per booking max: driver→host/spot and host→driver.
- Eligibility opens after booking completion + payment settled.
- Blind review window (e.g., 14 days):
  - Reviews hidden until both submit or window expires.
  - Reduces retaliatory behavior.
- Anti-extortion rules:
  - Detect messages containing coercion keywords; flag for trust review.
  - Remove reviews violating policy with auditable moderation reason.
- Weighting:
  - Verified/completed booking reviews only.
  - Recency-weighted aggregate with minimum review count threshold for display labels.

## 11) Implementation plan (epics → stories → tasks) with estimates and dependencies

### Epic 0 — Foundations (Week 1)
**Stories**
- Repo scaffolding (mobile/web/api/admin), shared types, env config.
- Docker Compose (Postgres+PostGIS, Redis, LocalStack/minio optional).
- CI baseline.

**Tasks checklist**
- [ ] Setup mono-repo (pnpm/turbo) and package boundaries.
- [ ] Add lint/typecheck/test scripts.
- [ ] Add OpenAPI generation + client SDK script.
- [ ] Create base infra configs and `.env.example`.

Estimate: 4–5 days. Dependency: none.

DoD: Fresh clone boots local stack and runs all checks.
Verification: `docker compose up`, `pnpm lint`, `pnpm test`, health checks.

### Epic 1 — Auth, profiles, RBAC (Week 1–2)
**Stories**
- User registration/login/refresh/magic link.
- Driver profile + vehicles (plate storage encrypted).
- Host profile + payout onboarding placeholder.

**Tasks**
- [ ] JWT access/refresh with rotation and logout.
- [ ] Password hash + reset flow.
- [ ] Roles/permissions middleware.
- [ ] Profile APIs and UI screens.

Estimate: 6–7 days. Depends on Epic 0.

DoD: Driver/host can authenticate and manage profile/vehicle.
Verification: auth integration tests, RBAC tests, mobile happy-path QA.

### Epic 2 — Listings and media (Week 2–3)
**Stories**
- Create/edit listing with geocode/map pin.
- Availability schedule + pricing rules.
- Photo uploads + moderation state.

**Tasks**
- [ ] Geocode service integration + fallback manual pin.
- [ ] CRUD for listing/rules/pricing/schedule.
- [ ] Signed URL upload + image metadata persistence.
- [ ] Duplicate listing heuristic (same host+address hash).

Estimate: 7–8 days. Depends on Epic 1.

DoD: Host can publish valid listing with photos and schedule.
Verification: API integration + UI form e2e + storage smoke tests.

### Epic 3 — Search + availability engine (Week 3–4)
**Stories**
- Map/list search with filters and availability.
- Listing detail with pricing estimate.

**Tasks**
- [ ] PostGIS spatial query + indexes.
- [ ] Availability computation service.
- [ ] Filter facets and pagination.
- [ ] Caching hot queries in Redis.

Estimate: 6–7 days. Depends on Epic 2.

DoD: Drivers can find available spots accurately for selected times.
Verification: deterministic availability unit tests + load test (k6).

### Epic 4 — Booking + payments core (Week 4–6)
**Stories**
- Create booking with payment authorization.
- Cancellations, no-show, extensions, receipts.
- Stripe Connect onboarding and payout ledger.

**Tasks**
- [ ] Booking transaction with locking + idempotency.
- [ ] Stripe PaymentIntent and webhook reconciliation.
- [ ] Cancellation/refund rules engine.
- [ ] Extension and overstay billing logic.
- [ ] Receipt generation/email.

Estimate: 10–12 days. Depends on Epic 3.

DoD: End-to-end booking and money movement works with edge cases.
Verification: webhook replay tests, concurrency tests, payment sandbox scenarios.

### Epic 5 — Messaging + notifications (Week 6–7)
**Stories**
- Booking-bound chat.
- Push/email notifications for lifecycle events.

**Tasks**
- [ ] Conversation/message APIs and websocket/polling strategy.
- [ ] Profanity/contact masking filters.
- [ ] Block/report actions.
- [ ] Notification templates and event triggers.

Estimate: 5–6 days. Depends on Epic 4.

DoD: Participants can communicate safely and receive key updates.
Verification: messaging e2e tests, abuse rule tests.

### Epic 6 — Reviews + trust/safety basics (Week 7)
**Stories**
- Two-sided reviews with blind publish window.
- Reporting/disputes flows.

**Tasks**
- [ ] Review eligibility + one-review-per-side constraints.
- [ ] Blind window scheduler.
- [ ] Report/dispute ticket APIs.
- [ ] Fraud flags + admin queue integration.

Estimate: 4–5 days. Depends on Epic 4.

DoD: Review and report systems operate with anti-retaliation.
Verification: policy tests + manual moderation QA.

### Epic 7 — Admin console + audit + data rights (Week 8–9)
**Stories**
- Admin lookup and booking management.
- Refund/dispute tooling.
- Audit logs and GDPR/CCPA workflows.

**Tasks**
- [ ] Admin auth/MFA and RBAC.
- [ ] User/booking/dispute management UI.
- [ ] Audit log viewer/export.
- [ ] Data export/delete job pipelines.

Estimate: 6–7 days. Depends on Epics 4, 6.

DoD: Support can resolve core incidents with full audit trail.
Verification: role tests, data request integration tests, support runbook UAT.

### Epic 8 — Hardening + launch prep (Week 10–12)
**Stories**
- Performance tuning, monitoring, runbooks, app store readiness.

**Tasks**
- [ ] Load/perf testing and query optimization.
- [ ] SLO dashboards + alerts.
- [ ] Incident playbooks + on-call rotation.
- [ ] Beta cohort rollout and feedback loop.

Estimate: 8–10 days. Depends on all prior.

DoD: Beta-ready with release gates met.
Verification: staging signoff checklist and go/no-go review.

## 12) Verification plan: test strategy + acceptance criteria + release gates

### 12.1 Test strategy
- Unit: domain logic (pricing, cancellation, eligibility, fraud rules).
- Integration: API + DB + Redis + Stripe webhook handlers.
- E2E:
  - Driver booking flow.
  - Host listing/publish flow.
  - Admin refund/dispute flow.
- Concurrency tests: simultaneous booking attempts same slot.
- Security tests: authz bypass, IDOR, rate-limit, injection.

### 12.2 Acceptance criteria (MVP)
- Zero confirmed double-bookings in automated concurrency suite.
- Successful booking/payment completion >95% in staging scripted runs.
- Refund/dispute workflows complete with auditable state changes.
- GDPR export/delete requests complete within defined SLA in staging.

### 12.3 Release gates
- All P0 tests passing; no open critical vulnerabilities.
- Observability dashboards and alerts live.
- Support runbook + escalation matrix approved.
- Legal/policy pages linked in apps (ToS/privacy/liability disclaimers ready).

### 12.4 Minimal production-minded CI/CD
- PR pipeline: lint, typecheck, unit tests, build.
- Main pipeline: integration tests, migration check, OpenAPI diff check.
- Staging deploy: auto on main; run smoke e2e.
- Production deploy: manual approval + canary (10%) + rollback automation.

## 13) Observability plan: logging/metrics/tracing + alerts
- Structured logs with correlation IDs (`request_id`, `user_id`, `booking_id`).
- Metrics:
  - API latency/error by endpoint.
  - Booking funnel counters.
  - Payment success/failure/refund rates.
  - Queue depth and job failures.
- Tracing: OpenTelemetry across API, DB, Stripe calls.
- Alerts (PagerDuty/Slack):
  - Booking error rate >2% 5m.
  - Stripe webhook failures >1%.
  - DB CPU >85% 15m or replication lag.
  - Push/email delivery failures spikes.

## 14) Rollout plan: beta → GA, migration/seed data, customer support ops

### 14.1 Beta to GA
- Phase 1: internal dogfood.
- Phase 2: invite-only beta (1 city, capped hosts/drivers).
- Phase 3: public beta (waitlist + referral).
- Phase 4: GA in first city; expand city-by-city.

### 14.2 Migration and seed data strategy
- Schema migrations via Prisma Migrate with backward-compatible steps.
- Seed scripts:
  - 50 hosts, 200 listings with realistic geospatial distribution.
  - 500 drivers + vehicles.
  - Booking history for analytics and moderation training.
- Data quality checks post-seed (constraints, geospatial validity).

### 14.3 Local dev setup (Docker Compose)
- Services: `postgres-postgis`, `redis`, `minio`, `mailhog`.
- Scripts:
  - `pnpm dev:infra` start services.
  - `pnpm db:migrate && pnpm db:seed`.
  - `pnpm dev` for API/mobile/web/admin.

### 14.4 Customer support ops
- Tiered queues: payments, access issues, safety incidents.
- Macros/playbooks for host cancel, blocked spot, no-show, overstay.
- SLA targets and escalation paths to risk/legal.

## 15) Risk register + mitigations (legal/regulatory, safety, fraud, platform rules)

| Risk | Impact | Likelihood | Mitigation |
|---|---:|---:|---|
| Local zoning/regulatory constraints on paid driveway rentals | High | Medium | City-by-city legal review, restricted launch zones, host attestations |
| Liability from vehicle damage/theft claims | High | Medium | Clear ToS/disclaimers, incident flow, evidence capture, optional insurance roadmap |
| Fake/duplicate/stolen-photo listings | High | Medium | Image hash checks, address verification, manual review queue, host risk scoring |
| Payment fraud/chargebacks | High | Medium | 3DS where needed, velocity rules, payout holds/reserves, dispute evidence automation |
| Last-minute host cancellations harm trust | High | Medium | Automatic refunds + host penalties + ranking suppression |
| GPS inaccuracies causing wrong arrivals | Medium | High | Arrival tolerance radius, pin correction UX, support-assisted adjustments |
| Review extortion/retaliation | Medium | Medium | Blind review window, coercion detection, moderation policy |
| Platform account takeover | High | Low | MFA for admins, anomalous login detection, token rotation |
| Accessibility gaps | Medium | Medium | WCAG audits, accessibility QA checklist in release gates |
| Store policy violations (payments/content) | High | Low | Policy review before submission, moderation and reporting mechanisms |

## Appendix A) Autonomous backlog checklists (agent-executable)
- [ ] Bootstrap mono-repo and shared packages.
- [ ] Provision docker services and env templates.
- [ ] Implement auth/profile/vehicle APIs + UI.
- [ ] Implement listing creation pipeline with media upload.
- [ ] Implement PostGIS search APIs + map views.
- [ ] Implement booking transaction + Stripe flows.
- [ ] Implement messaging and notification events.
- [ ] Implement review/report/dispute subsystems.
- [ ] Implement admin console and audit trails.
- [ ] Implement GDPR export/delete jobs.
- [ ] Add monitoring dashboards + alert policies.
- [ ] Run beta readiness checklist and release.

## Appendix B) Schema migration plan + example SQL

### Migration strategy
1. Create foundational tables (`users`, `profiles`, `listings`, `bookings`, `payments`).
2. Add constraints/indexes (including exclusion constraint for bookings).
3. Add messaging/reviews/admin/audit modules.
4. Backfill/seed and enable feature flags progressively.

### Example SQL (core)
```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email CITEXT UNIQUE NOT NULL,
  password_hash TEXT,
  role_flags TEXT[] NOT NULL DEFAULT ARRAY['driver'],
  status TEXT NOT NULL DEFAULT 'active',
  kyc_status TEXT NOT NULL DEFAULT 'unverified',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE listings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  host_user_id UUID NOT NULL REFERENCES users(id),
  title TEXT NOT NULL,
  description TEXT,
  address_line1 TEXT NOT NULL,
  city TEXT NOT NULL,
  region TEXT NOT NULL,
  postal_code TEXT NOT NULL,
  country_code TEXT NOT NULL,
  geom GEOGRAPHY(POINT, 4326) NOT NULL,
  pin_accuracy_m INTEGER NOT NULL DEFAULT 20,
  is_active BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_listings_geom ON listings USING GIST (geom);

CREATE TABLE availability_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  listing_id UUID NOT NULL REFERENCES listings(id) ON DELETE CASCADE,
  day_of_week SMALLINT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
  start_local_time TIME NOT NULL,
  end_local_time TIME NOT NULL,
  tz TEXT NOT NULL
);

CREATE TABLE bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  listing_id UUID NOT NULL REFERENCES listings(id),
  driver_user_id UUID NOT NULL REFERENCES users(id),
  start_ts TIMESTAMPTZ NOT NULL,
  end_ts TIMESTAMPTZ NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('pending','confirmed','active','completed','cancelled','disputed','no_show')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (end_ts > start_ts)
);

-- Prevent overlapping active bookings for same listing.
ALTER TABLE bookings
  ADD CONSTRAINT no_overlapping_bookings
  EXCLUDE USING GIST (
    listing_id WITH =,
    tstzrange(start_ts, end_ts, '[)') WITH &&
  ) WHERE (status IN ('pending','confirmed','active'));

CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID UNIQUE NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
  stripe_payment_intent_id TEXT UNIQUE NOT NULL,
  auth_amount_cents INTEGER NOT NULL,
  captured_amount_cents INTEGER,
  currency CHAR(3) NOT NULL DEFAULT 'USD',
  status TEXT NOT NULL CHECK (status IN ('requires_payment_method','requires_capture','succeeded','canceled','failed')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
  reviewer_user_id UUID NOT NULL REFERENCES users(id),
  reviewee_user_id UUID NOT NULL REFERENCES users(id),
  rating SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
  body TEXT,
  state TEXT NOT NULL DEFAULT 'pending_publish',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (booking_id, reviewer_user_id)
);

CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
  sender_user_id UUID NOT NULL REFERENCES users(id),
  body_filtered TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
