# Community Library — Claude Code Project Memory

Community Library is a monorepo for a local book-sharing platform (lend + donate physical books).
Stack: **.NET 10** API · **Angular** web (mobile-first) · **Swift/SwiftUI** iOS · **Podman Compose** infra · **Keycloak** identity · **PostgreSQL + Redis** data · **OpenTelemetry + Grafana** observability.

---

## Repo structure

```
api/          .NET 10 modular monolith (Api / Application / Domain / Infrastructure)
web/          Angular app
ios/          Swift/SwiftUI app
infra/
  compose/app/           app stack (api, web)
  compose/deps/          long-lived deps (keycloak, postgres, redis)
  compose/observability/ Grafana stack (alloy, tempo, loki, mimir, grafana)
  compose/prod/          production stack (traefik + all services)
  keycloak/              realm export and seed scripts
schemas/
  openapi/    generated OpenAPI spec + client SDKs (committed, do not edit by hand)
  db/         EF Core migration snapshots
docs/         all specification and reference docs (see table below)
.github/
  copilot-instructions.md   AI coding guardrails (read this)
  workflows/                CI workflows (to be scaffolded)
tools/        local dev scripts
```

## Essential docs — read before working in an area

| Doc | When to read |
|-----|-------------|
| `docs/spec.md` | Always — single source of truth for all decisions |
| `docs/uiux-spec.md` | Frontend / design work |
| `docs/api-endpoints.md` | Any API change |
| `docs/db-schema.md` | Any schema or query change |
| `docs/repo-organization.md` | CI, branching, CODEOWNERS |

## Context scoping with workspaces

Open the matching `.code-workspace` file to focus Claude's context on one area:

| Working on | Open this workspace |
|---|---|
| API (`.NET`) | `com-lib-api.code-workspace` |
| Web (Angular) | `com-lib-web.code-workspace` |
| iOS (SwiftUI) | `com-lib-ios.code-workspace` |

Each workspace includes `docs/` and the relevant `schemas/openapi/` folder so contracts stay visible.

## Local dev quickstart

```bash
cp infra/compose/observability/.env.example infra/compose/observability/.env
cp infra/compose/app/.env.example           infra/compose/app/.env
cp infra/compose/deps/.env.example          infra/compose/deps/.env
make dev-up   # networks + observability + deps + app
```

Grafana: http://localhost:3000 (admin/admin) · API: :8080 · Web: :4200 · Keycloak: :8081

## Critical invariants

- **Loan state machine** (fixed): `requested → approved → handover_pending → on_loan → return_pending → returned`. Cancellation terminals: `canceled_rejected | canceled_expired | canceled_user`. Every transition is logged in `loan_events`.
- **Donation state machine** (fixed): `offered → accepted → handover_pending → completed`. Terminals: `declined | canceled | expired`. Events logged in `donation_events`.
- **Short codes are lookup-only** — they never authorise transfers. Rate-limited + audited.
- **All state-changing API calls** require an `Idempotency-Key` header (24 h TTL).
- **All errors** use RFC 7807 Problem Details (`application/problem+json`).
- **No secrets in git.** Always start from `.env.example`.

## MVP — what is out of scope

Android · email/push notifications · group chats · fuzzy search · public reviews · GPS proximity · renewals · advanced moderation UI.

## AI coding guardrails

Full rules for architecture, state machines, security, code style, and testing are in:
**`.github/copilot-instructions.md`** — applies to all AI tools (Copilot, Claude Code, Claude.ai, Cursor, etc.)
