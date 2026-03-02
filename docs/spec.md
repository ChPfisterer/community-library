# Community Library — Comprehensive Specification (MVP)

This document consolidates all agreed decisions for the MVP. It is the single source of truth and links to deeper reference docs in `docs/`.

## 1. Product scope (MVP)
- Purpose: Enable local communities to share physical books via lending and donations.
- Platforms: Web (Angular) and iOS (Swift/SwiftUI). Android deferred.
- Key capabilities:
  - Scan a book (EAN‑13) to import edition; create personal copies.
  - Print labels with QR + short code; scan to hand over/return.
  - Request/approve/confirm loan; reminders and overdue handling.
  - Donate books to existing users; ownership transfer.
  - 1:1 text‑only messaging with read receipts.
  - Search by title/author/ISBN‑13; optional owner filter.
  - Offline: capture scans and queue actions; sync later.
  - In‑app notifications only (email/push later).
- Non‑goals for MVP: groups, media sharing, fuzzy search, public discovery maps, moderation tools, email/push delivery, Android.

## 2. Architecture overview
- Backend: .NET 10, ASP.NET Core (Minimal APIs), Modular Monolith (Api, Application, Domain, Infrastructure).
- Data: PostgreSQL; EF Core (Npgsql). Redis for cache/queues. SignalR for realtime.
- Identity: Keycloak (OIDC). Self‑registration ON. Login by username or email.
- Containers: Docker Compose (or Podman Compose); two stacks (app and observability) bridged by external network `telemetry-net`.
- Observability: OpenTelemetry → Alloy → Tempo (traces), Loki (logs), Mimir (metrics); Grafana dashboards.
- Docs & contracts: RFC 7807 errors; Idempotency-Key for state changes; correlation headers.

References
- Database schema: `docs/db-schema.md`, ER: `docs/db-er.mmd`, UML: `docs/db-uml.puml`.
- Endpoints mapping: `docs/api-endpoints.md`.
- AI coding guardrails: `.github/copilot-instructions.md`.

## 3. Identity and auth
- Canonical user identifier: Keycloak `sub`.
- Roles: `user`, `moderator`, `admin` (policy‑based auth in API). MVP focuses on `user`.
- Username (nickname): 3–20 chars, regex `^[A-Za-z0-9._]{3,20}$`, case‑insensitive unique. Optional display name may include spaces/emojis.
- Email verification: OFF in dev, ON in prod.

## 4. QR, short code, labels
- QR payload (static): `c` (bookCopyId, GUID), `v` (qrVersion, int), optional `kid` (keyId), `s` (HMAC‑SHA256 base64url over `c.v`).
- Owner can re‑issue a label (increments `v`), invalidating old labels.
- Short code: 12 chars (Crockford Base32, excludes I,L,O,U) + 1 Crockford mod-37 check character = 13 total; grouped `AAA-BBB-CCC-DDD-X`. The check character catches single-character errors and adjacent transpositions.
  - Storage: canonical form is 13 uppercase chars with no hyphens (`varchar(13)`); hyphens are display-only. API strips hyphens and uppercases input before lookup/insert.
  - Persist short code → (bookCopyId, qrVersion); deactivate old codes on reissue; regenerate on collision.
  - Input: accept case‑insensitive; hyphens optional; ignore whitespace; display uppercase grouped `AAA-BBB-CCC-DDD-X`.
- Label printing: PDF for Avery L4770/5160 (A4/Letter). Include QR + grouped short code text.

## 5. Lending and donations
- Loan state machine: `requested → approved → handover_pending → on_loan → return_pending → returned`.
  - Cancellation: either party (borrower or owner) may cancel in `requested` or `approved` states → `canceled_user`; logged with `actor_user_id` to distinguish who cancelled.
  - Auto‑expiry: `requested` after 72h, `handover_pending` after 24h → `canceled_expired`.
  - Owner rejection: owner rejects a `requested` loan → `canceled_rejected`.
  - Default loan duration: 4 weeks. Reminders: due T‑48h/T‑12h; overdue +3d/+7d/+14d; mark‑lost at +14d.
  - Events: every transition logged in `loan_events`.
  - Handover/return method: `method` on `start` declares the initiator's intent; `method` on `confirm` records the actual method used. The two are independent — QR can fall back to OTP at confirm time without needing to restart. Each is recorded separately in `loan_events` for audit.
- Book copy status: `active` (default) ↔ `archived` (owner-driven, reversible) ↔ `lost` (set by mark-lost; recoverable via activate when book is physically returned). Both `archived` and `lost` suppress from search and block new loans/donations. Transition `archived → lost` is blocked (copy not in circulation).
- Donations: two flows, both requiring existing registered users only.
  - **Targeted**: owner picks a specific recipient at offer time → recipient accepts or declines → handover confirm.
  - **Community**: owner posts an open offer (discoverable in search with `for_donation` badge) → interested users register interest → owner picks one recipient → handover confirm.
  - Both flows converge at `accepted → handover_pending → completed`; dual‑scan or OTP fallback for handover.
  - On confirm: `book_copies.owner_user_id` transfers to recipient; full event history kept in `donation_events`.
  - Auto‑expiry: targeted offers expire 72h after creation (no response); community offers expire 7d after creation (fixed from `created_at`, regardless of interest registrations).
  - Owner may cancel any offer before `handover_pending`.
  - Interested users may withdraw their interest while the donation is in `offered` state (before owner picks); withdrawal is logged in `donation_events` as `withdraw`.

## 6. Messaging (1:1)
- Text only, max 2,000 chars.
- Read receipts ON; explicit only — client calls `POST /messages/{id}/read` when the message is visible on screen. Fetching messages (`GET /threads/{id}/messages`) does not auto-mark as read; background syncs must not trigger read receipts. Edit window 2 min; each edit appends the prior body to `message_edits` (append-only, retained 30 days for abuse review).
- Delete for me anytime; delete for both within 2 min if unread (tombstone shown).
- Block/report supported (moderation enforcement deferred).

## 7. Search and privacy
- Search fields: title, author, ISBN‑13; optional owner exact nickname filter.
- Availability filter: `for_loan` (shareable copies), `for_donation` (active community offers), `any` (both, default).
- Sort: relevance, recently added.
- Fuzzy/typo tolerance deferred.
- Privacy: Only shareable books appear in `for_loan` results; only community donation offers appear in `for_donation` results. Targeted donations are not discoverable. Personal metadata private‑by‑default; opt‑in when shared.

## 8. Offline and sync
- Fully offline: view My Library, capture scans, queue actions (pending book, handover/return intents), draft messages.
- State changes apply only after server confirmation.
- Sync triggers: on resume, manual, periodic.
- Reconciliation: last‑writer‑wins for personal metadata using server-assigned timestamp (not client time, which is unreliable); idempotency keys for loan events.

## 9. Notifications
- In‑app only for MVP (no email/push). Persist notifications with timestamps; later quiet‑hours semantics.
- Donation notification events: `donation_invite` (targeted recipient), `donation_interest_received` (donor gets when someone expresses interest), `donation_picked` (interested user gets when selected), `donation_declined` (donor gets when targeted recipient declines), `donation_completed` (recipient gets on ownership transfer).

## 10. Observability
- API emits OTLP traces/metrics/logs to Alloy on `telemetry-net`; routed to Tempo/Mimir/Loki; Grafana dashboards.
- Include `trace_id`/`span_id` in logs. Sampling: 10% head, always sample errors and slow (>500ms) requests.

## 11. API conventions
- Errors: RFC 7807 with `error_code` and `field_errors`.
- Idempotency: Require `Idempotency-Key` for state‑changing operations (loan request/approve/confirm, donation confirm). TTL 24h; return original response on replay.
- Correlation: Echo `traceparent` and `x-request-id`; include trace info in logs.

## 12. Data model summary
- Entities: editions, edition_snapshots, book_copies, qr_labels, short_codes, loans, loan_events, donations, donation_interests, donation_events, threads, messages, message_edits, shelves, shelf_items, reviews, notifications, audit_short_code_lookups, otp_events.
- See diagrams and schema docs for columns, constraints, and relationships.

## 13. Endpoint surface (MVP)
- See `docs/api-endpoints.md`. Covers: editions import/search, copy creation/reissue, QR sign/verify, short‑code lookup + audit, lending lifecycle, OTP fallback, donations, messaging, notifications, admin audit.

## 14. Security considerations
- No PII in QR/short codes; codes are lookup‑only with strict rate limits and audit logging.
- HMAC keys rotation plan (tracked in OPEN_TOPICS); use `kid` for selection.
- Enforce ownership and role checks for all state changes.
- Limit OTP attempts; 10‑minute validity.
- Validate inputs strictly (regex for usernames, Crockford Base32 for short codes).

## 15. Rate limits and audit
- Rate limits on short‑code lookups; log actor, timestamp, device info.
- Log all loan/donation events and message edits/deletes for abuse review.

## 16. Non‑functional requirements
- Availability: single node acceptable for MVP; horizontal scale later.
- Performance: p95 API <300ms typical; always trace slow (>500ms) for debugging.
- Data retention: 7‑day rolling for observability; app data persistent.
- Internationalization: EN‑US only for MVP (German-market launch; localization post-MVP).
- Accessibility: WCAG AA minded; best effort for MVP.
- GDPR (required for German/EU market launch):
  - Lawful basis: contract performance for core features (lending, messaging, donations); no separate consent needed for functional data.
  - Right to erasure: anonymize `user_id` in event/audit tables using tombstone UUID; hard-delete personal content (reviews, shelves, messages, notifications); delete Keycloak account. Procedure implemented before go-live.
  - Right to data portability: `GET /me/export` returns all user data as JSON. Implemented in Phase 12.
  - Privacy policy and terms of service: published before go-live.
  - Infrastructure: Hetzner (EU/German host); self-hosted Keycloak. No personal data leaves the EU.
  - No analytics cookies without consent; suppress product analytics until a consent mechanism is in place.

## 17. Out of scope (MVP)
- Android client, media attachments, group chats, fuzzy search, public discovery map, email/push delivery, advanced moderation tooling, renewals, public review retrieval (reviews schema supports `visibility=public` for future use; no read endpoint in MVP).

## 18. Open topics
- Tracked in `docs/OPEN_TOPICS.md`.
