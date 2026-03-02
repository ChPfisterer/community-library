# Open Topics Backlog (to discuss / defer)

This file tracks decisions deferred beyond MVP or items needing further clarification. Keep in sync with README and `.github/copilot-instructions.md` when resolved.

## Architecture & Platform
- Work layer (canonical Works) above Editions: data model, migration plan from Editions-only.
- API versioning and deprecation policy for public endpoints.
- Potential microservices split post-MVP: service boundaries (catalog, loans, messaging, notifications).

## Identity & Security
- Apple Sign In (Keycloak IdP) and any other social IdPs (Google/Apple/…); rollout plan.
- 2FA policy (TOTP/SMS) enforcement (optional vs required by role) and recovery flows.
- Email verification strictness and exceptions (dev/test realms, staging toggles).

## QR, Short Code, and Printing
- Key rotation strategy for QR signing secrets (`kid` lifecycle, decommission plan).
- Additional label templates/sizes (e.g., small spine labels, thermal/BLE printers like Brother/QZ Tray).
- Deterministic short code alternative (HMAC-derived) vs stored mapping trade-offs (keep stored mapping for MVP).

## Lending & Donations
- Proximity enforcement for code-based handover/return (BLE/GPS/SSID) — currently OFF for MVP.
- Renewals and extensions policy for loans (limits, approvals, reminders adjustment).
- Donations to non-users (lite signup vs require existing user) — currently users-only.

## Messaging & Moderation
- Group chats and expanded features (typing indicators, presence, attachments/media).
- Abuse detection/filters (profanity, spam) and automated actions.
- Moderator/admin UI and enforcement actions (mute, suspend, delete content); audit trails and SLAs.

## Search & Discovery
- Fuzzy/typo tolerance and ranking improvements (PG trigram now, external engine later: Meilisearch/OpenSearch).
- Location-aware discovery (GPS/geofence/proximity) and privacy rules.
- Public reviews: `reviews.visibility` field exists for future use; define public review retrieval endpoint and visibility rules post-MVP.

## Offline & Sync
- Background sync cadence tuning; battery/network heuristics.
- Conflict resolution beyond LWW (e.g., CRDTs) for certain personal metadata.
- Local storage quotas and purging strategy (iOS/Web specifics).

## Notifications & Comms
- Email provider integration (Postmark/Resend), templates, quiet-hour semantics; later push/SMS.
- In-app notification center UX (batching, read/unread, preferences).

## Observability & Ops
- Grafana dashboards catalog (API golden signals, QR/short-code funnels, Keycloak health).
- Retention and sampling tuning per environment; alert thresholds and on-call runbooks.
- Secure remote access patterns for Grafana in prod (SSO/Keycloak integration).

## Data & Integrations
- GDPR/CCPA compliance: **GDPR is required before go-live (German/EU market)**. Core decisions resolved in spec.md §16 and roadmap cross-cutting stories. Remaining open items: CCPA applicability (defer unless US users targeted), long-term data retention policy per category, formal DPA with Hetzner.
- Data export formats (user library, messages): covered by GET /me/export endpoint (Phase 12).
- Recommendations system scope and data sources (later phase).
- Additional metadata sources beyond OL/GB/ISBNDb; field-level merge heuristics.

## Delivery & Tooling
- Podman Compose skeletons for app/observability stacks, including `telemetry-net` bootstrap.
- Database schema migrations (EF Core) and seed data strategy.
- CI/CD pipelines, environments (dev/staging/prod), secrets management.

## Metrics & Product Analytics
- Expand success metrics (D1/D30, WAU/MAU, loan completion time) and add experiments framework later.
- Consider adding a product analytics tool (e.g., PostHog) if Grafana dashboards become limiting.
