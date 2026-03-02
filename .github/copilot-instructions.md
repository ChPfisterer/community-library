# AI Coding Guardrails — Community Library

These instructions apply to all AI coding assistants working in this repository:
**GitHub Copilot, Claude Code, Claude.ai, Cursor, and any other AI-assisted tools.**

Read `docs/spec.md` before making any architectural or data-model decisions. It is the single source of truth.

---

## Architecture

- Backend is a **Modular Monolith** with Clean Architecture. Layer order (innermost → outermost): `Domain → Application → Infrastructure → Api`. Inner layers must never reference outer ones.
- `Domain` holds entities and pure business rules — no EF Core, no HTTP, no external packages.
- `Application` holds use cases / command handlers — orchestrates domain logic, no direct HTTP concerns.
- `Infrastructure` holds EF Core, Redis, Keycloak client, external HTTP calls.
- `Api` (ASP.NET Core Minimal APIs) only wires up HTTP concerns and delegates to `Application`.
- Do not add Android support — it is explicitly out of scope for MVP.

## State machines

The loan and donation state machines are **fixed by spec**. Do not add, remove, or short-circuit states.

**Loan states:** `requested → approved → handover_pending → on_loan → return_pending → returned`
Cancellation terminals: `canceled_rejected`, `canceled_expired`, `canceled_user`

**Donation states:** `offered → accepted → handover_pending → completed`
Terminals: `declined`, `canceled`, `expired`

Every transition must be logged in the corresponding `loan_events` / `donation_events` table.

## API conventions

- All errors: **RFC 7807 Problem Details** (`Content-Type: application/problem+json`). Fields: `type`, `title`, `status`, `detail`, `instance`, `error_code`. Validation errors add `field_errors[]`.
- State-changing operations (loan request/approve/confirm, donation confirm) require an **`Idempotency-Key`** header. TTL 24 h; replay returns the original response; conflicting replay returns 409.
- Responses include **W3C `traceparent`** and **`x-request-id`**; logs must include `trace_id` / `span_id`.
- 429 responses include `Retry-After` seconds.

## Data & security

- **Never commit secrets**, tokens, credentials, or certificates. Use `.env` files (always start from `.env.example`) or Podman secrets.
- **No PII in QR codes or short codes.** Short codes are lookup-only and never authorise state changes.
- QR payloads must include an HMAC-SHA256 signature (`s` field). Verify before acting on any scan.
- Short-code lookups are rate-limited (5/min per IP, 50/day per device) and fully audited.
- OTP codes are single-use, 10-minute expiry, bound to `loanId` and direction. Max 3 active per loan/hour.
- Enforce ownership and role checks (`user`, `moderator`, `admin`) on every state-changing endpoint.
- GDPR (required for EU/German launch): right to erasure via tombstone UUIDs; right to export via `GET /me/export`. Never remove these flows.

## Database

- All schema changes require a corresponding **EF Core migration** under `schemas/db/`.
- Do not write raw SQL outside of EF Core unless using `FromSqlRaw` for a documented performance reason.
- Follow the naming conventions in `docs/db-schema.md` (snake_case table and column names, timestamptz for all timestamps).
- Do not drop or rename columns without a migration strategy — the DB may have live data.

## Code style

| Area | Rules |
|------|-------|
| `.NET / C#` | Minimal APIs. Serilog for logging. No Newtonsoft.Json (use `System.Text.Json`). Async all the way down. |
| `Angular` | Mobile-first. Follow existing component and service structure. No jQuery. Reactive forms. |
| `Swift / iOS` | SwiftUI only — no UIKit unless there is genuinely no SwiftUI equivalent. Combine or async/await for concurrency. |
| `All` | New packages (NuGet / npm / SPM) require explicit approval before introduction. |

## Testing

- New business logic in `Domain` or `Application` must have unit tests.
- New API endpoints should have integration tests using the test project conventions.
- Prefer in-memory providers over mocking EF Core.
- Do not write tests that only assert mocks were called — test observable behaviour.

## Monorepo boundaries

- Work in the area relevant to the task: `api/`, `web/`, `ios/`, `infra/`, `docs/`.
- Do not introduce cross-area import dependencies outside of `schemas/openapi/` contracts.
- Generated OpenAPI clients live in `web/src/generated/` and `ios/Sources/Generated/` — do not edit them by hand.

## What not to implement (MVP scope)

| Out of scope | Reason |
|---|---|
| Android client | Explicitly deferred post-MVP |
| Email / push notifications | In-app only for MVP |
| Fuzzy / typo-tolerant search | Deferred post-MVP |
| Group chats | Out of scope |
| Public review retrieval | Schema supports it; no read endpoint in MVP |
| Advanced moderation UI | Reports stored, enforcement deferred |
| GPS / proximity search | City/ZIP text filter only |
| Renewals | Not in spec |
