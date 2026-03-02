# API Endpoints to Tables Mapping (MVP)

Conventions
- Auth: All endpoints require OIDC Bearer tokens from Keycloak unless stated.
- Errors: RFC 7807 Problem Details (application/problem+json) with `error_code` and optional `field_errors`.
- Idempotency: For state-changing endpoints marked [IDEMPOTENT], clients send `Idempotency-Key`.
- Correlation: Echo `traceparent` and `x-request-id` in responses. Logs include `trace_id`/`span_id`.

## Catalog & Editions

GET /editions/{isbn13}
- Reads: editions, edition_snapshots (latest)
- Purpose: Fetch edition details

GET /editions?q=title|author|isbn13&availability=for_loan|for_donation|any&owner={nickname?}&sort=relevance|recent&page&size
- Reads: editions, book_copies, donations
- Purpose: Search editions; filter by availability (for_loan: shareable copies, for_donation: active community offers, any: both); optional owner nickname filter

POST /editions/import [IDEMPOTENT]
- Body: { isbn13 }
- Writes: editions (upsert), edition_snapshots (insert)
- Purpose: Import edition metadata by ISBN (from OL→GB→ISBNDb)

## Ownership & Labeling

POST /book-copies [IDEMPOTENT]
- Body: { isbn13 }
- Writes: book_copies (insert), qr_labels (version=1), short_codes (version=1)
- Purpose: Create a book copy for the current user and issue initial QR + short code

POST /book-copies/{id}/qr/reissue [IDEMPOTENT]
- Writes: qr_labels (version++ insert), short_codes (new code, prior inactive)
- Purpose: Re-issue label (rotate version and short code)

POST /book-copies/{id}/archive [IDEMPOTENT]
- (no body; caller must be owner; only allowed when status=active and no active loan or donation)
- Writes: book_copies (status=archived)
- Purpose: Owner removes copy from circulation temporarily; suppresses from search and blocks new loans/donations

POST /book-copies/{id}/activate [IDEMPOTENT]
- (no body; caller must be owner; allowed when status=archived or status=lost)
- Writes: book_copies (status=active)
- Purpose: Owner returns a copy to active circulation; covers archived copies and books physically recovered after being marked lost (prior loan record is unchanged)

GET /short-codes/{code}
- Reads: short_codes (active), book_copies, editions
- Writes: audit_short_code_lookups (insert)
- Purpose: Lookup a book by handwritten/printed short code (lookup-only)

## QR Sign/Verify (server-side)

POST /qr/sign [internal]
- Body: { c: bookCopyId, v: version, kid? }
- Reads: book_copies
- Purpose: Sign QR payload (used by label generator)

POST /qr/verify
- Body: { c, v, kid?, s }
- Reads: book_copies, qr_labels
- Purpose: Verify label at scan time (defense-in-depth)

## Lending

GET /loans?role=owner|borrower&state={state?}&page&size
- Reads: loans, book_copies, editions
- Purpose: List loans for the current user; role defaults to all (owner + borrower); optionally filter by state

GET /loans/{id}
- Reads: loans, loan_events, book_copies, editions
- Purpose: Get a single loan with its full event history

POST /loans/request [IDEMPOTENT]
- Body: { book_copy_id, due_at? }
- Writes: loans (insert state=requested), loan_events(request)
- Purpose: Borrower requests a loan

POST /loans/{id}/approve [IDEMPOTENT]
- Body: { due_at }
- Writes: loans (state=approved, due_at), loan_events(approve)
- Purpose: Owner approves

POST /loans/{id}/reject [IDEMPOTENT]
- Writes: loans (state=canceled_rejected), loan_events(reject)
- Purpose: Owner rejects

POST /loans/{id}/handover/start [IDEMPOTENT]
- Body: { method: "qr"|"otp" }
- Writes: loans (state=handover_pending), loan_events(handover_start)
- Purpose: Either party initiates handover

POST /loans/{id}/handover/confirm [IDEMPOTENT]
- Body: { method: "qr"|"otp", otp_code? }
- Writes: loans (state=on_loan), loan_events(handover_confirm), otp_events(validate)
- Purpose: Confirm handover via dual-scan or OTP fallback; method here is independent of the method used in handover/start (QR intent may fall back to OTP at confirm time)

POST /loans/{id}/return/start [IDEMPOTENT]
- Body: { method: "qr"|"otp" }
- Writes: loans (state=return_pending), loan_events(return_start)
- Purpose: Initiate return

POST /loans/{id}/return/confirm [IDEMPOTENT]
- Body: { method: "qr"|"otp", otp_code? }
- Writes: loans (state=returned), loan_events(return_confirm), otp_events(validate)
- Purpose: Confirm return via dual-scan or OTP fallback; method is independent of return/start method

POST /loans/{id}/cancel [IDEMPOTENT]
- (no body; caller must be borrower or owner; only allowed in requested or approved states)
- Writes: loans (state=canceled_user), loan_events(cancel)
- Purpose: Either party withdraws before handover begins; actor_user_id in loan_events identifies who cancelled

POST /loans/{id}/expire
- Writes: loans (state=canceled_expired), loan_events(auto_expire)
- Purpose: System job to expire requested (72h) or handover_pending (24h)

POST /loans/{id}/mark-lost [IDEMPOTENT]
- Writes: loans (state=on_loan retained), book_copies (status=lost), loan_events(mark_lost)
- Purpose: Owner marks lost after 14 days overdue (no compensation tracking)

## OTP (Handover/Return fallback)

POST /loans/{id}/otp/generate [IDEMPOTENT]
- Writes: otp_events(generate, loan_id set)
- Purpose: Generate 6-digit one-time code (10 min) for loan handover/return

POST /loans/{id}/otp/validate [IDEMPOTENT]
- Body: { code }
- Writes: otp_events(validate, loan_id set)
- Purpose: Validate OTP for loan handover/return (used by loan confirm endpoints above)

POST /donations/{id}/otp/generate [IDEMPOTENT]
- Writes: otp_events(generate, donation_id set)
- Purpose: Generate 6-digit one-time code (10 min) for donation handover

POST /donations/{id}/otp/validate [IDEMPOTENT]
- Body: { code }
- Writes: otp_events(validate, donation_id set)
- Purpose: Validate OTP for donation handover (used by donations handover/confirm above)

## Donations

GET /donations?role=donor|recipient&state={state?}&page&size
- Reads: donations, book_copies, editions
- Purpose: List donations for the current user; role defaults to all (donor + recipient); optionally filter by state

GET /donations/{id}
- Reads: donations, donation_events, book_copies, editions
- Purpose: Get a single donation with its full event history

POST /donations/offer [IDEMPOTENT]
- Body: { book_copy_id, type: "targeted"|"community", recipient_user_id? (required if targeted) }
- Writes: donations (insert state=offered), donation_events(offer)
- Purpose: Owner creates a targeted donation (specific recipient) or a community offer (discoverable in search)

-- Targeted flow --

POST /donations/{id}/accept [IDEMPOTENT]
- (no body; caller must be the targeted recipient)
- Writes: donations (state=accepted), donation_events(accept)
- Purpose: Targeted recipient accepts the offer

POST /donations/{id}/decline [IDEMPOTENT]
- (no body; caller must be the targeted recipient)
- Writes: donations (state=declined), donation_events(decline)
- Purpose: Targeted recipient declines the offer

-- Community flow --

POST /donations/{id}/interest [IDEMPOTENT]
- (no body; caller must not be the donor)
- Writes: donation_interests (insert), donation_events(interest)
- Purpose: Interested user registers interest in a community offer

DELETE /donations/{id}/interest [IDEMPOTENT]
- (no body; caller must be the user who registered interest; only allowed while donation state=offered)
- Writes: donation_interests (delete), donation_events(withdraw)
- Purpose: User withdraws their interest before the donor picks a recipient

GET /donations/{id}/interests [donor only]
- Reads: donation_interests (with user nicknames)
- Purpose: Donor views the list of interested users to choose from

POST /donations/{id}/pick [IDEMPOTENT]
- Body: { recipient_user_id }
- Writes: donations (recipient_user_id set, state=accepted), donation_events(pick)
- Purpose: Donor selects a recipient from the interested users list

-- Shared handover (both flows converge here) --

POST /donations/{id}/handover/start [IDEMPOTENT]
- Body: { method: "qr"|"otp" }
- Writes: donations (state=handover_pending), donation_events(handover_start)
- Purpose: Donor or recipient initiates the physical handover

POST /donations/{id}/handover/confirm [IDEMPOTENT]
- Body: { method: "qr"|"otp", otp_code? }
- Writes: donations (state=completed), book_copies (owner_user_id = recipient_user_id), donation_events(handover_confirm)
- Purpose: Confirm handover via dual-scan or OTP fallback; ownership transfers to recipient

-- Management --

POST /donations/{id}/cancel [IDEMPOTENT]
- (no body; caller must be donor; only allowed before handover_pending)
- Writes: donations (state=canceled), donation_events(cancel)
- Purpose: Donor withdraws the offer

POST /donations/{id}/expire
- Writes: donations (state=expired), donation_events(expire)
- Purpose: System job; expires offers based on fixed timer from created_at (targeted: 72h, community: 7d)

## Messaging (1:1)

GET /threads
- Reads: threads (for current user)

POST /threads [IDEMPOTENT]
- Body: { other_user_id }
- Writes: threads (insert or fetch existing by user pair)

GET /threads/{thread_id}/messages?before|after&page&size
- Reads: messages

POST /threads/{thread_id}/messages [IDEMPOTENT]
- Body: { body }
- Writes: messages (insert)

POST /messages/{id}/edit [IDEMPOTENT]
- Body: { body }
- Writes: messages (edited_at, body)

POST /messages/{id}/delete-for-both [IDEMPOTENT]
- Writes: messages (tombstone via flags)

POST /messages/{id}/delete-for-me [IDEMPOTENT]
- Writes: messages (deleted_for_sender or recipient)

POST /messages/{id}/read [IDEMPOTENT]
- Writes: messages (read_at)

## Notifications

GET /notifications?unread_only=true|false&page&size
- Reads: notifications

POST /notifications/{id}/read [IDEMPOTENT]
- Writes: notifications (read_at)

## Short code audits

GET /admin/audit/short-codes?code&from&to&page&size [admin]
- Reads: audit_short_code_lookups

## User Data (GDPR)

GET /me/export
- Reads: book_copies, loans, donations, threads, messages, shelves, reviews, notifications
- Purpose: Data portability — returns all user data as JSON (GDPR right to portability; required before go-live)

POST /me/erase [admin-confirmed]
- Writes: anonymize actor_user_id/user_id in loan_events, donation_events, donation_interests, message_edits, audit_short_code_lookups, otp_events (replace with tombstone UUID 00000000-0000-0000-0000-000000000001); hard-delete reviews, shelves, shelf_items, messages, notifications; trigger Keycloak account deletion
- Purpose: GDPR right to erasure (required before go-live)

## Notes
- Admin/moderation enforcement is deferred for MVP; report collection will be added later.
