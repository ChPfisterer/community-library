# Roadmap — Phased Delivery with Small Stories

Use this roadmap to plan epics, create small user stories, and track progress in a GitHub Project. Durations are indicative and can overlap with sufficient capacity.

Legend
- [E] Epic, [S] Story, [T] Task, [D] Deliverable, [DEP] Dependency
- Mark [IDEMPOTENT] where clients must send Idempotency-Key.

## Phase 0 — Project scaffolding (0.5–1 week)
- [S] Repo hygiene
  - [T] LICENSE, CODE_OF_CONDUCT, CONTRIBUTING, SECURITY policy ✅
  - [T] PR template, Issue templates (bug, story), labels
  - [T] Branch protection, required reviews, status checks
- [S] CI
  - [T] GitHub Actions: build api/web/ios jobs (matrix), lint, unit tests ✅
  - [T] Code coverage upload
- [S] Infra skeleton
  - [T] Podman Compose: app stack (api, web, keycloak, postgres, redis) ✅
  - [T] Observability stack (alloy, tempo, loki, mimir, grafana) on `telemetry-net` ✅
  - [T] Seed scripts for Keycloak realm/clients/roles
- [S] Local test environment (Parallels Ubuntu VM)
  - [T] Full Compose stack on VM; verify Linux-specific behaviour (Podman socket, networking)
  - [T] Confirm Compose configs match cloud behaviour before first staging deploy
- [S] Cloud staging environment (Hetzner)
  - [T] Provision Hetzner server; configure firewall, SSH keys, DNS A record for staging domain
  - [T] Deploy test Compose stack (app + deps); validate Traefik TLS routing
  - [T] Deploy observability stack; label all metrics/logs with `env=staging`
  - [T] Wire CI: push to main triggers deploy to staging, runs smoke tests
  - [T] Provision Keycloak staging realm (email verification OFF; mirrors dev config)

## Phase 1 — Identity & shell (1–2 weeks)
- [E] Keycloak integration
  - [S] Provision dev realm
  - [S] Configure clients: api (confidential), web (public), ios (public)
  - [S] Login by username/email; self‑registration; email verification OFF in dev
- [E] API shell (.NET 10)
  - [S] Solution structure: Api/Application/Domain/Infrastructure
  - [S] Health endpoints; OpenTelemetry bootstrap; Serilog with trace ids
  - [S] Postgres + EF Core baseline; Redis; SignalR hub
- [E] Web shell (Angular)
  - [S] OIDC login with Keycloak; basic layout; protected routes
- [E] iOS shell (SwiftUI)
  - [S] AppAuth login, Keychain tokens; simple tab nav

## Phase 2 — Catalog & copies (1–2 weeks)
- [E] Editions import/search
  - [S] API: POST /editions/import [IDEMPOTENT]
  - [S] API: GET /editions/{isbn13}, GET /editions (search)
  - [S] Web/iOS: Scan EAN‑13, show match; manual ISBN entry
- [E] Book copies
  - [S] API: POST /book-copies [IDEMPOTENT]
  - [S] My Library list (availability badges)

## Phase 3 — Labels: QR + short code (1 week)
- [E] Labeling
  - [S] API: POST /book-copies/{id}/qr/reissue [IDEMPOTENT]
  - [S] API: GET /short-codes/{code} + audit insert
  - [S] Label generator: PDF (A4/Letter) with QR + grouped code
  - [S] Web/iOS: Short code input UX (case-insensitive, hyphens optional)

## Phase 4 — Lending core (2–3 weeks)
- [E] State machine & events
  - [S] API: POST /loans/request [IDEMPOTENT]
  - [S] API: POST /loans/{id}/approve, /reject [IDEMPOTENT]
  - [S] API: POST /loans/{id}/handover/start [IDEMPOTENT]
  - [S] API: POST /loans/{id}/handover/confirm [IDEMPOTENT]
  - [S] API: POST /loans/{id}/return/start [IDEMPOTENT]
  - [S] API: POST /loans/{id}/return/confirm [IDEMPOTENT]
  - [S] System job: /loans/{id}/expire
  - [S] Owner action: /loans/{id}/mark-lost [IDEMPOTENT]
  - [S] Web/iOS flows: borrower and owner screens, state transitions

## Phase 5 — OTP fallback (handover/return) (0.5–1 week)
- [E] OTP
  - [S] API: /loans/{id}/otp/generate [IDEMPOTENT]
  - [S] API: /loans/{id}/otp/validate [IDEMPOTENT]
  - [S] Web/iOS: 6-digit code UX; timers and retry limits

## Phase 6 — Messaging (1–2 weeks)
- [E] Threads & messages
  - [S] API: threads list/create; messages list/create/edit/delete/read
  - [S] SignalR: message delivery + read receipts
  - [S] Web/iOS: chat UI, 2-minute edit/delete rules

## Phase 7 — Notifications (0.5–1 week)
- [E] In-app notifications
  - [S] API: list + mark as read [IDEMPOTENT]
  - [S] Emit notifications for loan lifecycle/donations
  - [S] Web/iOS: badge counts, inbox

## Phase 8 — Observability (0.5–1 week)
- [E] Dashboards and alerts
  - [S] OTLP wiring to Alloy; Grafana dashboards (API latency, errors, DB)
  - [S] Log correlation (trace ids); sampling policies

## Phase 9 — Offline & sync (1–2 weeks)
- [E] Client offline queues
  - [S] Local persistence; queued actions (scans/intents/messages)
  - [S] Sync manager: resume/manual/periodic; LWW for personal data

## Phase 10 — Donations (1–2 weeks)
- [E] Targeted donation
  - [S] API: POST /donations/offer (type=targeted) [IDEMPOTENT]
  - [S] API: POST /donations/{id}/accept, /decline [IDEMPOTENT]
  - [S] Web/iOS: offer flow (pick recipient), accept/decline UX
- [E] Community donation
  - [S] API: POST /donations/offer (type=community) [IDEMPOTENT]
  - [S] API: POST /donations/{id}/interest [IDEMPOTENT]
  - [S] API: GET /donations/{id}/interests; POST /donations/{id}/pick [IDEMPOTENT]
  - [S] Web/iOS: community offer UX; interest list for donor; pick flow
  - [S] Search: availability=for_donation filter + badge in results
- [E] Shared handover & management (reuse OTP patterns from Phase 5)
  - [S] API: handover/start, handover/confirm, cancel, expire [IDEMPOTENT]
  - [S] Notifications: donation_invite, donation_interest_received, donation_picked, donation_declined, donation_completed

## Phase 11 — Search polish (0.5 week)
- [E] Search UX & ranking tweaks
  - [S] Owner filter; relevance vs recent; no fuzzy in MVP

## Phase 12 — Production go-live (0.5–1 week)
- [E] Production deployment (Hetzner, same server as staging)
  - [S] Prod Compose stack: separate network from staging; Traefik routes prod domains
  - [S] Shared observability stack: staging and prod both ship to same Alloy/Grafana; differentiate by `env=prod` label
  - [S] Keycloak staging realm: enable email verification ON (requires email provider configured); validate full registration + email verification flow end-to-end on staging before prod flip
  - [S] Keycloak prod realm: email verification ON; hardened client config; prod redirect URIs
  - [S] Secrets audit: all credentials via `.env` (not committed); document rotation procedure
  - [S] DB backup and restore procedure: scheduled dumps, tested restore
  - [S] Monitoring alerts: error rate, p95 latency, disk usage thresholds in Grafana
  - [S] On-call runbook stub: restart procedures, rollback steps, escalation contacts
  - [S] Go-live smoke test checklist: register, add book, lend, donate, message, check traces in Grafana
  - [S] GDPR pre-launch checklist: erasure endpoint tested, data export verified, privacy policy live, cookie consent active

## Cross-cutting stories
- [S] Security hardening: input validation, authZ policies, rate limits
- [S] Data migrations: EF Core migrations + rollback plan
- [S] Test harness: API integration tests; UI e2e smoke
- [S] Performance budgets and basic load test
- [S] GDPR compliance (required before go-live — German/EU market):
  - Anonymization procedure for right-to-erasure: anonymize user_id in event/audit tables with tombstone UUID; hard-delete personal content
  - API: POST /me/erase (triggers erasure + Keycloak account deletion) [admin-confirmed]
  - API: GET /me/export (data portability — returns all user data as JSON)
  - Privacy policy and terms of service published
  - Cookie consent mechanism (suppress analytics until consent given)
  - Lawful basis documented per processing activity

## Issue/story drafting template
Title: [S] Short, action-oriented title
Description:
- Context
- Acceptance criteria (Gherkin optional)
- API/DB touchpoints (link to endpoints + tables)
- Out of scope
- QA notes and test data

## Dependencies map (high level)
- Phase 0 staging story is prerequisite for CI deploy pipeline in Phase 1.
- Phase 1 is prerequisite for all feature phases.
- Phase 2 precedes 3–5 and 10.
- Phase 4 precedes 5 and 7.
- Phase 5 (OTP) precedes Phase 10 (donations reuse OTP patterns).
- Phase 6 independent once identity ready.
- Phase 12 requires Phases 1–11 substantially complete and staging validated.

## Deliverables per phase
- Working endpoints (per `docs/api-endpoints.md`)
- UI flows for web and iOS
- Tests (unit/integration), basic docs and demo video
- Grafana dashboard updates (where applicable)
