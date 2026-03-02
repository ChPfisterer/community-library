# Database Schema (MVP)

This schema outlines the minimum tables and relationships for the Community Library MVP. User identity is provided by Keycloak; `user_id` below refers to the Keycloak `sub` (UUID as string).

Notes
- Timestamps are UTC (`timestamptz`).
- All FKs `ON UPDATE CASCADE ON DELETE RESTRICT` unless stated.
- JSONB fields hold normalized snapshots or event metadata with explicit schemas enforced at the application layer.
- GDPR erasure exception: `user_id`/`actor_user_id` columns in event and audit tables (`loan_events`, `donation_events`, `donation_interests`, `message_edits`, `audit_short_code_lookups`, `otp_events`) must support anonymization on right-to-erasure requests. These columns must be nullable OR accept a well-known tombstone UUID (`00000000-0000-0000-0000-000000000001`) so historical records can be anonymized without breaking referential integrity. Personal content tables (`reviews`, `shelves`, `shelf_items`, `notifications`, `messages`) are hard-deleted on erasure.

## Core catalog

editions
- isbn13 varchar(13) PK
- title text NOT NULL
- authors text[] NOT NULL DEFAULT '{}'
- publisher text NULL
- publish_date date NULL
- page_count int NULL
- language_code varchar(16) NULL
- cover_url text NULL
- sourced_from text NOT NULL DEFAULT 'openlibrary' -- openlibrary|google|isbndb|manual
- created_at timestamptz NOT NULL DEFAULT now()

Indexes
- (title)
- GIN (authors)

edition_snapshots
- id uuid PK DEFAULT gen_random_uuid()
- isbn13 varchar(13) NOT NULL REFERENCES editions(isbn13)
- fields jsonb NOT NULL -- normalized field/value snapshot
- source text NOT NULL -- which provider supplied this snapshot
- created_at timestamptz NOT NULL DEFAULT now()

## Ownership and labeling

book_copies
- id uuid PK DEFAULT gen_random_uuid()
- owner_user_id uuid NOT NULL -- Keycloak sub
- isbn13 varchar(13) NOT NULL REFERENCES editions(isbn13)
- status text NOT NULL DEFAULT 'active' -- active|archived|lost; active↔archived (owner-driven, reversible); active→lost (set by mark-lost); lost→active (activate endpoint, when physically recovered); archived→lost blocked
- shareable boolean NOT NULL DEFAULT false -- controls search visibility and loan requestability; opt-in
- created_at timestamptz NOT NULL DEFAULT now()
Indexes
- (shareable, isbn13) -- supports filtered search queries

qr_labels
- book_copy_id uuid NOT NULL REFERENCES book_copies(id)
- version int NOT NULL
- kid text NULL -- key id for HMAC rotation
- signature text NOT NULL -- base64url HMAC-SHA256 over `c.v`
- active boolean NOT NULL DEFAULT true
- created_at timestamptz NOT NULL DEFAULT now()
PK (book_copy_id, version)

short_codes
- code varchar(13) PK -- canonical form: 13 uppercase chars (12 Crockford Base32 + 1 checksum), no hyphens; API strips hyphens and uppercases on input before lookup/insert
- book_copy_id uuid NOT NULL REFERENCES book_copies(id)
- qr_version int NOT NULL
- checksum char(1) NOT NULL
- active boolean NOT NULL DEFAULT true
- created_at timestamptz NOT NULL DEFAULT now()
Constraints
- UNIQUE (book_copy_id, qr_version)
- When a new qr_version is issued, set previous rows for that book_copy_id to active=false

## Lending

loans
- id uuid PK DEFAULT gen_random_uuid()
- book_copy_id uuid NOT NULL REFERENCES book_copies(id)
- owner_user_id uuid NOT NULL
- borrower_user_id uuid NOT NULL
- state text NOT NULL -- requested|approved|handover_pending|on_loan|return_pending|returned|canceled_rejected|canceled_expired|canceled_user
- due_at timestamptz NULL -- set when approved
- created_at timestamptz NOT NULL DEFAULT now()
- updated_at timestamptz NOT NULL DEFAULT now()
Indexes
- (owner_user_id)
- (borrower_user_id)
- (state)

loan_events
- id uuid PK DEFAULT gen_random_uuid()
- loan_id uuid NOT NULL REFERENCES loans(id)
- type text NOT NULL -- request|approve|reject|cancel|handover_start|handover_confirm|return_start|return_confirm|auto_expire|mark_lost
- method text NULL -- qr|otp|system
- actor_user_id uuid NULL
- metadata jsonb NULL -- device info, warnings, etc.
- created_at timestamptz NOT NULL DEFAULT now()
Index
- (loan_id, created_at)

## Messaging (1:1)

threads
- id uuid PK DEFAULT gen_random_uuid()
- user_a_id uuid NOT NULL
- user_b_id uuid NOT NULL
- created_at timestamptz NOT NULL DEFAULT now()
Constraints
- UNIQUE (LEAST(user_a_id, user_b_id), GREATEST(user_a_id, user_b_id)) -- one thread per pair

messages
- id uuid PK DEFAULT gen_random_uuid()
- thread_id uuid NOT NULL REFERENCES threads(id)
- from_user_id uuid NOT NULL
- body text NOT NULL CHECK (char_length(body) <= 2000)
- edited_at timestamptz NULL
- deleted_for_sender boolean NOT NULL DEFAULT false
- deleted_for_recipient boolean NOT NULL DEFAULT false
- read_at timestamptz NULL
- created_at timestamptz NOT NULL DEFAULT now()
Indexes
- (thread_id, created_at)
- (from_user_id, created_at)

message_edits
- id bigserial PK
- message_id uuid NOT NULL REFERENCES messages(id)
- body text NOT NULL -- body as it was *before* this edit was applied
- edited_at timestamptz NOT NULL DEFAULT now()
Index
- (message_id, edited_at)
Notes
- Append-only; rows are never updated or deleted during the 30-day retention window.
- Purge rows older than 30 days on the same schedule as deleted message tombstones.

## Personal metadata

shelves
- id uuid PK DEFAULT gen_random_uuid()
- user_id uuid NOT NULL
- name text NOT NULL
- created_at timestamptz NOT NULL DEFAULT now()
Constraints
- UNIQUE (user_id, lower(name))

shelf_items
- shelf_id uuid NOT NULL REFERENCES shelves(id) ON DELETE CASCADE
- isbn13 varchar(13) NULL REFERENCES editions(isbn13)
- book_copy_id uuid NULL REFERENCES book_copies(id)
- created_at timestamptz NOT NULL DEFAULT now()
Constraints
- CHECK ((isbn13 IS NOT NULL) <> (book_copy_id IS NOT NULL)) -- exactly one of the two
PK (shelf_id, COALESCE(isbn13, '0'), COALESCE(book_copy_id, '00000000-0000-0000-0000-000000000000'))

reviews
- id uuid PK DEFAULT gen_random_uuid()
- user_id uuid NOT NULL
- isbn13 varchar(13) NOT NULL REFERENCES editions(isbn13)
- rating smallint NULL CHECK (rating BETWEEN 1 AND 5)
- text text NULL
- visibility text NOT NULL DEFAULT 'private' -- private|public
- created_at timestamptz NOT NULL DEFAULT now()
Constraints
- UNIQUE (user_id, isbn13) -- one review per user per edition

## Notifications

notifications
- id uuid PK DEFAULT gen_random_uuid()
- user_id uuid NOT NULL
- type text NOT NULL -- loan_requested|loan_approved|loan_declined|handover_pending|due_soon|due|overdue|return_confirmed|donation_invite|donation_interest_received|donation_picked|donation_declined|donation_completed
- payload jsonb NOT NULL -- structured event payload
- read_at timestamptz NULL
- created_at timestamptz NOT NULL DEFAULT now()
Indexes
- (user_id, created_at DESC)

## Donations

donations
- id uuid PK DEFAULT gen_random_uuid()
- book_copy_id uuid NOT NULL REFERENCES book_copies(id)
- donor_user_id uuid NOT NULL -- Keycloak sub
- recipient_user_id uuid NULL -- set at offer (targeted) or after owner picks (community)
- type text NOT NULL -- targeted|community
- state text NOT NULL DEFAULT 'offered' -- offered|accepted|handover_pending|completed|declined|canceled|expired
- created_at timestamptz NOT NULL DEFAULT now()
- updated_at timestamptz NOT NULL DEFAULT now()
Indexes
- (donor_user_id)
- (recipient_user_id)
- (state)
- (book_copy_id) -- enforce at most one active offer per copy at application layer

donation_interests
- id uuid PK DEFAULT gen_random_uuid()
- donation_id uuid NOT NULL REFERENCES donations(id)
- user_id uuid NOT NULL -- interested user (Keycloak sub)
- created_at timestamptz NOT NULL DEFAULT now()
Constraints
- UNIQUE (donation_id, user_id)

donation_events
- id uuid PK DEFAULT gen_random_uuid()
- donation_id uuid NOT NULL REFERENCES donations(id)
- type text NOT NULL -- offer|interest|withdraw|pick|accept|decline|handover_start|handover_confirm|cancel|expire
- method text NULL -- qr|otp|system
- actor_user_id uuid NULL
- metadata jsonb NULL -- device info, etc.
- created_at timestamptz NOT NULL DEFAULT now()
Index
- (donation_id, created_at)

## Auditing (security/rate limits)

audit_short_code_lookups
- id bigserial PK
- code varchar(13) NOT NULL -- canonical form (no hyphens), matches short_codes.code
- outcome text NOT NULL -- hit|miss|rate_limited
- hashed_ip text NULL
- hashed_device text NULL
- book_copy_id uuid NULL
- created_at timestamptz NOT NULL DEFAULT now()

otp_events
- id bigserial PK
- loan_id uuid NULL REFERENCES loans(id) -- set for loan handover/return OTP
- donation_id uuid NULL REFERENCES donations(id) -- set for donation handover OTP
- actor_user_id uuid NULL
- action text NOT NULL -- generate|validate|lock
- outcome text NOT NULL -- ok|invalid|expired|locked|rate_limited
- created_at timestamptz NOT NULL DEFAULT now()
Constraints
- CHECK ((loan_id IS NULL) <> (donation_id IS NULL)) -- exactly one must be set

## Reference enums (application-level)
- loan.state: requested|approved|handover_pending|on_loan|return_pending|returned|canceled_rejected|canceled_expired|canceled_user
- loan_events.type: request|approve|reject|cancel|handover_start|handover_confirm|return_start|return_confirm|auto_expire|mark_lost
- donations.type: targeted|community
- donations.state: offered|accepted|handover_pending|completed|declined|canceled|expired
- donation_events.type: offer|interest|withdraw|pick|accept|decline|handover_start|handover_confirm|cancel|expire
- notifications.type: loan_requested|loan_approved|loan_declined|handover_pending|due_soon|due|overdue|return_confirmed|donation_invite|donation_interest_received|donation_picked|donation_declined|donation_completed
