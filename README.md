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
- AI coding guardrails: `.github/copilot-instructions.md` · Claude Code memory: `CLAUDE.md`

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
- Use strong, unique passwords in `.env` (or Podman secrets). Never commit secrets — always start from `.env.example`.
- Consider systemd units or a process supervisor for auto-start on reboot.
- For multi-node HA or advanced rollouts, migrate to Kubernetes later.

## Technical reference

All design decisions live in `docs/`. The README is intentionally high-level.

| Document | Contents |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Claude Code project memory — repo map, workspace guide, critical invariants |
| [`.github/copilot-instructions.md`](.github/copilot-instructions.md) | AI coding guardrails for all tools (Copilot, Claude Code, Claude.ai, Cursor) |
| [`docs/spec.md`](docs/spec.md) | MVP technical specification — architecture, data model, API conventions, GDPR, security |
| [`docs/uiux-spec.md`](docs/uiux-spec.md) | UI/UX specification — personas, screens, components, flows (complements `spec.md`) |
| [`docs/api-endpoints.md`](docs/api-endpoints.md) | Endpoint surface: methods, paths, auth, request/response shapes |
| [`docs/db-schema.md`](docs/db-schema.md) | Database schema with columns, constraints, and indexes |
| [`docs/db-er.mmd`](docs/db-er.mmd) | ER diagram (Mermaid) |
| [`docs/db-uml.puml`](docs/db-uml.puml) | UML diagram (PlantUML) |
| [`docs/roadmap.md`](docs/roadmap.md) | Phased roadmap with small stories — pick one to contribute |
| [`docs/project-tracking.md`](docs/project-tracking.md) | GitHub Project setup, issue templates, labels, DoD |
| [`docs/repo-organization.md`](docs/repo-organization.md) | Monorepo layout, CI/CD blueprint, branching, CODEOWNERS |
| [`docs/workspaces.md`](docs/workspaces.md) | Editor setup by area — VS Code workspaces (API/Web) and Xcode (iOS) |
| [`docs/versions.md`](docs/versions.md) | Pinned container image versions and update policy |
| [`docs/branch-protection.md`](docs/branch-protection.md) | Branch protection rules and merge strategy |
| [`docs/OPEN_TOPICS.md`](docs/OPEN_TOPICS.md) | Design questions still under discussion |
