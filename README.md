Why Community Library?
Because the best books aren’t locked in warehouses — they’re already on our shelves. Community Library makes it easy to share physical books with people around you, with just the right mix of light structure, privacy, and a friendly UX.

## What you can do
- Scan, add, and organize your books
	- Add by barcode (EAN‑13/ISBN‑13) or manual entry; metadata fetched automatically.
	- Create personal copies and shelves. Your personal notes stay private by default.

- Label once, share forever
	- Print smart labels with a QR code and a human‑friendly short code.
	- Re‑issue labels any time; old ones are safely invalidated.

- Borrowing, without the awkwardness
	- Request, approve, and confirm handover with a quick dual scan (or a one‑time code if you can’t scan).
	- Built‑in reminders before due dates; gentle nudges when overdue.

- Donate with a tap
	- Offer a book to an existing user and confirm the transfer in seconds.

- Chat 1:1 to coordinate
	- Lightweight, text‑only messaging with read receipts and a short edit window.

- Search what’s shareable
	- Find books by title, author, or ISBN‑13; optionally filter by owner nickname.

- Works offline
	- Capture scans, queue actions, and sync when you’re back online.

## How it works (in 60 seconds)
1) Add your book by scanning its retail barcode.  
2) Print a label (QR + short code) and stick it in the book.  
3) A neighbor requests your book → you approve.  
4) Meet and confirm handover by scanning each other’s labels (or use a one‑time 6‑digit code).  
5) Get reminders before it’s due, and confirm the return the same way.

## Privacy by default
- No personal data in QR or short codes. Short codes are lookup‑only and rate‑limited with audit trails.
- Your personal notes and shelves are private unless you choose to share.
- Identity and login handled by Keycloak; you can sign in with username or email.

## Screenshots
Coming soon. We’ll add previews as the MVP UI lands for Web and iOS.

## Roadmap highlights
- Now → MVP: catalog import, labels (QR + short code), lending, donations, 1:1 messaging, in‑app notifications, offline queue.
- Next: polish search and UX, dashboards/observability, and a steady cadence of quality‑of‑life improvements.
- See the full breakdown with small stories in `docs/roadmap.md`.

## Get involved
- Star the repo to follow progress.
- Open a small story from `docs/roadmap.md` using the “Story” issue template.
- Use feature branches (e.g., `feat/web/short-code-lookup`) and open a PR. CI is path‑filtered, so only your area builds.

## For developers (at a glance)
- Stack: .NET 10 (API) · Angular (Web) · Swift/SwiftUI (iOS) · Postgres · Redis · Keycloak · SignalR · OpenTelemetry + Grafana stack
- Monorepo with path‑filtered CI. OpenAPI contracts live in `schemas/openapi/`.
- Focus your editor and AI context with per‑area workspaces:
	- `com-lib-api.code-workspace` · `com-lib-web.code-workspace` · `com-lib-ios.code-workspace`
- Start with the spec and diagrams:
	- MVP spec: `docs/spec.md`
	- Endpoints ↔ tables: `docs/api-endpoints.md`
	- DB schema & diagrams: `docs/db-schema.md`, `docs/db-er.mmd`, `docs/db-uml.puml`
	- Repo & CI/CD: `docs/repo-organization.md`

## Project status
We’re building the MVP in the open. The spec is stable for MVP; code scaffolding is next. If you want to help bootstrap the API/Web/iOS shells or infra compose stacks, pick a Phase 0/1 story from `docs/roadmap.md` and say hello in a PR.

—

Looking for the technical deep‑dive? See `docs/spec.md` (single source of truth) and the rest of the `docs/` folder.

## Try it locally (preview)

Spin up placeholder stacks now; we’ll swap in real API/Web services as they land.

### Prerequisites
- Podman (or Docker as an alternative)

### 1) Configure environment
Create .env files (you can keep defaults):

```bash
cp infra/compose/observability/.env.example infra/compose/observability/.env
cp infra/compose/app/.env.example infra/compose/app/.env
cp infra/compose/deps/.env.example infra/compose/deps/.env
```

Create the shared external networks (once):

```bash
podman network create telemetry-net
podman network create comlib-app-net
```

Alternatively, use the Makefile shortcuts:

```bash
# Create networks and bring everything up (observability + deps + app)
make dev-up

# Bring stacks up/down individually
make obs-up
make deps-up
make app-up
make app-down

# Tear down all dev stacks and keep networks
make dev-down

# Remove the external networks when you're done
make networks-rm
```

### 2) Start observability (optional but recommended)
Grafana, Tempo, Loki, Mimir, Alloy:

```bash
podman compose -f infra/compose/observability/docker-compose.yml up -d
```

Open Grafana: http://localhost:3000 (admin/admin)

### 3) Option A — Split stacks (recommended)
Run long‑lived dependencies separately, iterate on app quickly.

Start dependencies once:

```bash
podman compose -f infra/compose/deps/docker-compose.yml up -d
```

Start app (API/Web):

```bash
podman compose -f infra/compose/app/docker-compose.yml up -d
```

What’s running
- Web (placeholder): http://localhost:${WEB\_HTTP\_PORT:-4200}
- API (placeholder): http://localhost:${API\_HTTP\_PORT:-8080}
- Keycloak (dev): http://localhost:${KEYCLOAK\_HTTP\_PORT:-8081} (admin/admin)
- Postgres: localhost:${POSTGRES\_TCP\_PORT:-5432} (postgres/postgres)
- Redis: localhost:${REDIS\_TCP\_PORT:-6379}

Stop/restart just the app while keeping deps running:

```bash
podman compose -f infra/compose/app/docker-compose.yml down
podman compose -f infra/compose/app/docker-compose.yml up -d
```

### 4) Build real services (when available)
Once the API/Web projects are added with Containerfiles (or Dockerfiles), build local images:

```bash
# API (example)
podman compose -f infra/compose/app/docker-compose.yml build api

# Web (example)
podman compose -f infra/compose/app/docker-compose.yml build web
```

Restart the stack to use your built images:

```bash
podman compose -f infra/compose/app/docker-compose.yml up -d --build
```

### 5) Tear down

```bash
# Stop and remove containers (app stack)
podman compose -f infra/compose/app/docker-compose.yml down

# Stop and remove containers (observability)
podman compose -f infra/compose/observability/docker-compose.yml down

# If you used the split deps stack, stop it too
podman compose -f infra/compose/deps/docker-compose.yml down

# Remove the shared networks (if not used elsewhere)
podman network rm telemetry-net || true
podman network rm comlib-app-net || true
```

Note: API/Web are placeholders to validate the stack. We’ll replace them with real services and add environment wiring (Keycloak realm, DB migrations, OpenTelemetry) as the implementation lands.

## Production deployment (compose on a single host)

For MVP, a single-server Podman Compose deployment is sufficient. We include a production stack that terminates TLS with Traefik and routes traffic to Web, API, Keycloak, and (optionally) Grafana.

### Prerequisites
- A Linux VM or server with Podman and podman-compose plugin
- DNS A/AAAA records for your domains (e.g., app.example.com, api.example.com, auth.example.com)
- Ports 80/443 open to the internet

### Configure environment

```bash
cp infra/compose/prod/.env.example infra/compose/prod/.env
# Edit infra/compose/prod/.env to set domains, emails, passwords, and image tags
```

### Bring up the edge + app stack

```bash
podman compose -f infra/compose/prod/docker-compose.yml up -d
```

This starts:
- Traefik (TLS, :80/:443) with Let’s Encrypt
- Web (at https://$WEB\_HOST)
- API (at https://$API\_HOST)
- Keycloak (at https://$AUTH\_HOST)
- Postgres, Redis (internal only)

### Optional: bring up observability

```bash
podman compose -f infra/compose/prod/docker-compose.observability.yml up -d
```

Grafana will be reachable at https://$GRAFANA\_HOST. Secure it with an admin password and network policies.

### Updating images

```bash
podman compose -f infra/compose/prod/docker-compose.yml pull
podman compose -f infra/compose/prod/docker-compose.yml up -d
```

### Teardown

```bash
podman compose -f infra/compose/prod/docker-compose.yml down
podman compose -f infra/compose/prod/docker-compose.observability.yml down
```

Notes
- Use strong, unique passwords in .env (or Podman secrets). Don't commit secrets.
- Consider systemd units or a process supervisor for auto-start on reboot.
- For multi-node HA or advanced rollouts, migrate to Kubernetes later.

### Identity & Access
- Identity Provider: Keycloak (self-registration enabled).
- Login: username (unique nickname) OR email; email verification OFF in dev, ON in prod.
- Roles: user, moderator, admin (policy-based authorization in API).
 - Username (nickname) policy (MVP):
 	\- Length: 3–20 characters; case-insensitive uniqueness.
 	\- Allowed: letters, numbers, underscore, dot. Regex: `^[A-Za-z0-9._]{3,20}$`.
 	\- No spaces or other symbols. Separate optional display name can include spaces/emojis.

### Platform & Stack
- Backend: .NET 9 (C#) Modular Monolith with Clean Architecture (Api, Application, Domain, Infrastructure).
- Web: Angular (mobile-first). iOS: native Swift/SwiftUI.
- Data: PostgreSQL + Redis (rate limits, codes, pub/sub).
- Containerization: Podman Compose split stacks:
	- App stack: api, web
	- Deps stack: keycloak, postgres, redis
	- Observability stack: grafana, mimir, loki, tempo, alloy (Grafana Agent)
- Networking: shared external networks `telemetry-net` (observability) and `comlib-app-net` (app/deps);
	API, Web, and Keycloak are exposed to the host in dev.

### Observability
- OTLP traces/metrics/logs → Alloy → Tempo (traces), Mimir (metrics), Loki (logs); dashboards/alerts in Grafana.
- .NET: OpenTelemetry for ASP.NET Core, HttpClient, EF Core, SignalR; Serilog with trace/span IDs.
- Keycloak: /metrics scraped; logs to Loki; audit events enabled.
- Retention (MVP): Traces 7d, Logs 14d, Metrics 30d. Head sampling \~10%; always sample errors and slow (\>500ms) requests.

### Global Metadata & Catalog
- Editions model only for MVP (keyed by ISBN-13). Work layer deferred.
- Metadata sources: Open Library → Google Books → ISBNDb. Persist normalized snapshots and provenance.
- Add books via: EAN-13/ISBN-13 scan, ISBN entry, manual. Offline “pending book” allowed; fetch metadata on next sync.

### QR Labels, Short Codes, and Fallbacks
- QR payload (static label):
	- c: bookCopyId (opaque UUID)
	- v: qrVersion (int)
	- kid: keyId (optional; for secret rotation)
	- s: signature = base64url(HMAC-SHA256(secret[kid], c + "." + v))
- Short code: 12 chars (Crockford Base32) + 1 checksum (total 13), grouped 3-3-3-3 + checksum (e.g., ABC-DEF-GHI-JKL-X); rotates with qrVersion; can be printed OR handwritten in the book.
- Short code is lookup-only (never authorizes transfers). Rate-limited and fully audited.
- Handover/return defaults: dual QR scan; in-app fallback via one-time 6‑digit code (10 min expiry), no proximity requirement; all events logged (actors, method, timestamps, device info).
 - Short code input rules (MVP): display uppercase; user input is case-insensitive; hyphens optional; whitespace ignored; Crockford Base32 alphabet (digits 0–9 and letters A–Z excluding I, L, O, U).

#### Short Code Storage (MVP)
- Approach: stored mapping in DB for clarity and auditing.
- Table (suggested): `short_codes(code pk, book_copy_id fk, qr_version int, checksum char(1), active bool, created_at timestamptz)`
	- Constraints: `unique(code)`, `unique(book_copy_id, qr_version)`, `active` false for older versions on re-issue.
	- Collision handling: if generated code collides, regenerate until unique (extremely rare with 12 Base32 + checksum).
	- Rotation: re-issuing a label (qrVersion++) generates a new short code and marks prior mappings inactive.

### Label Printing (MVP)
- Label content: QR code + human-friendly short code text (grouped Base32 with checksum) + optional deep link.
- Paper/templates: A4 and US Letter; Avery-compatible (e.g., A4 L4770, US 5160).
- Size: standard address label (\~63.5×38.1 mm / 2.625×1 in).
- Output: downloadable PDF sheets (multiple labels) and a single-label print option; include cut marks/margins.
- Regeneration: re-issuing a label bumps qrVersion, regenerates the QR and rotates the short code.

### Lending/Borrowing Lifecycle
- States: requested → approved → handover\_pending → on\_loan → return\_pending → returned (closed).
- Cancellations: canceled\_rejected, canceled\_expired (auto), canceled\_user.
- Computed flags: due\_soon, overdue.
- Dual-scan at handover and return; fallback via one-time code when QR isn’t possible.
- Timers & reminders (in-app only for MVP; email deferred. When email is enabled later, respect quiet hours 22:00–07:00):
	- Default loan duration: 4 weeks (owner may set custom due date at approval).
	- Request auto-expire: 72h without owner action.
	- Handover\_pending auto-expire: 24h (can re-initiate without re-approval).
	- Due soon: T‑48h and T‑12h (borrower); Due at: immediate reminder.
	- Overdue reminders: +3d, +7d, +14d (borrower); owner nudges at +7d, +14d.
	- Mark as lost: allowed at 14d overdue (no compensation tracking in MVP).

### Donations & Ownership Transfer (MVP)
- Donation is a distinct flow (not a loan): owner marks a book copy as "Available to donate"; recipient accepts.
- Eligibility: donations only to existing registered users (recipient must be logged in).
- Confirmation: same dual-confirm pattern as loans
	- Preferred: QR dual-scan
	- Fallback: one-time 6‑digit code (10 min), no proximity requirement, fully audited
- On confirmation: ownership of the bookCopy transfers to the recipient; transfer history is kept on the bookCopy.
- Personal metadata does not transfer; recipient starts with a clean slate.

### Messaging (MVP)
- 1:1 text-only (max 2,000 chars). Delivery via SignalR; offline outbox retry.
- Read receipts enabled (can be toggled later).
- Edit: 2‑minute window with “edited” indicator; server keeps originals for abuse review.
- Delete-for-me: anytime. Delete-for-both: within 2 minutes if unread (tombstone shown).
- Block/report supported; content scanning deferred (manual reports only).
- Retention: unlimited by default; logical deletes retained 30 days for abuse investigations.

### Moderation (MVP)
- User-level: block/report available in clients.
- Backend: accept and store reports with basic metadata (reporter, target, reason, timestamps).
- Admin enforcement: deferred (no moderator actions/UI in MVP). Reports are for later review.
- Evidence retention: deleted/edited message originals retained up to 30 days, then purged.

### Notifications (MVP)
- Channels: In-app only. Email/SMS deferred.
- Events covered: loan requested/approved/declined, handover pending, due soon, due, overdue (+3d/+7d/+14d), return confirmed, donation invitation/accepted.
- Delivery semantics: queue-and-retry; deduplicate within 1h window if state already changed.

### Metrics & Analytics (MVP)
- Success metrics
	- Activation: ≥3 books added by a new user within 7 days of signup.
	- Core value: ≥1 completed loan per active user within 30 days.
	- Retention: 7‑day rolling retention for users who added ≥1 book (any meaningful activity on days 2–7).
- Minimal instrumentation (server-side)
	- Events: user\_signed\_up, app\_open/session\_start, book\_added (isbn13), loan\_requested/approved/handover\_confirmed/return\_confirmed.
	- IDs: user\_id (Keycloak sub), loan\_id, device\_platform; timestamps server-side.
- Dashboards
	- Activation funnel (signed\_up → added\_1 → added\_3), loans per user and completion rate, D1/D7 cohort chart.
	- Implement in Grafana using counters/time series from API; upgrade to dedicated product analytics later if needed.

### API Conventions (MVP)
- Errors: RFC 7807 Problem Details (content-type: application/problem+json)
	- Fields: type, title, status, detail, instance
	- Extensions: error\_code (string), field\_errors[] for validation (field, message, code)
	- 429 includes Retry-After seconds; 409 for illegal state transitions (e.g., handover without approval)
	- Example (validation 400):
	  {
	```
	"type": "https://docs.comlib/errors/validation",
	"title": "Validation failed",
	"status": 400,
	"detail": "One or more fields are invalid.",
	"field_errors": [{"field": "isbn13", "message": "Invalid checksum", "code": "isbn_checksum"}]
	```
	  }
- Idempotency: send Idempotency-Key header for state-changing endpoints (loan request/approve/confirm, donation accept/confirm)
	- Server stores request hash + result for 24h; replays return the original response
	- Conflicting replay (different body with same key) returns 409 Problem
- Correlation: responses include W3C traceparent and x-request-id; logs include trace\_id/span\_id for cross-system tracing

### Search & Discovery (MVP)
- Search fields: title, author, ISBN-13; optional owner nickname exact filter.
- Filters: availability (available vs on-loan). Location: city/ZIP text filter (no GPS proximity).
- Sorting: relevance (simple scoring), recently added.
- Implementation: PostgreSQL (GIN/trigram ready), but fuzzy/typo tolerance deferred.
- Privacy: only books explicitly marked shareable are discoverable; personal metadata never appears in global results.

### Offline & Sync
- Fully offline: view My Library (cached), capture scans, queue actions (pending book, handover/return intents), draft messages.
- Server-required: state changes (handover/return) apply only after server confirmation.
- Sync triggers: app resume, manual refresh, periodic while foregrounded (e.g., every 15 min).
- Conflict policy: last-writer-wins for personal metadata; idempotency keys for loan events.
- Local cache cap: \~50–100 MB with LRU eviction for old snapshots/images.

### Privacy Defaults
- Personal metadata (notes, ratings, reviews, shelves) is opt-in and private-by-default.
- Sharing reviews is per-item opt-in; when shared, visible to community; can be changed later.

### Barcode/Scanner Support (MVP)
- iOS: AVFoundation live scanner.
- Supported: QR (app labels), EAN‑13/ISBN‑13 (retail barcodes). Others deferred.

### Rate Limits & Audit (MVP)
- Short code lookup
	- 5 attempts/min per IP/device; 50 attempts/day per device. Exponential backoff on failures.
	- Per-code guard: lock a specific short code for 5 minutes after 10 consecutive failures from the same IP/device.
	- Audit log: timestamp, outcome (hit/miss), hashed IP, hashed device fingerprint, and resolved bookCopyId if found.
- One-time handover/return code (6 digits)
	- Lifetime: 10 minutes; single-use; bound to loanId and direction (handover vs return).
	- Generation: max 3 active codes per loan per hour; only loan participants can generate.
	- Validation: 3 attempts/min per loan per device; hard lock for 5 minutes after 10 failures.
	- Audit log: timestamp, actor userIds, method=otp, device info, outcome. All events logged.
