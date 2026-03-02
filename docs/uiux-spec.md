# Community Library — UI/UX Specification

This specification is aimed at design and frontend development. It complements the technical specification (`spec.md`) and describes product context, personas, use cases, screens, components, simplified data models, and API drafts.

---

## 1. Product description

**Community Library** is a local book-sharing platform for communities. Users can lend and donate physical books within their neighborhood or group.

**Key features**
- Import a book by EAN-13 scan and create personal copies
- Print QR labels with a short code and attach them to books
- Lending process with digital handover proof (QR scan or OTP)
- Donate books to specific users or openly to the community
- Direct messages between lenders and borrowers
- Full-text search by title, author, ISBN with availability filter
- Offline mode: scans and actions are saved locally and synced when connectivity is restored
- In-app notifications for all relevant events

**Platforms:** Web (Angular, mobile-first) · iOS (SwiftUI)
**Target market:** Germany / EU (GDPR-compliant)
**Out of scope for MVP:** Android, email/push notifications, group chats, public reviews, fuzzy search

---

## 2. User personas

### Persona A — Lisa, the book owner
- **Age:** 34 · **Occupation:** Teacher
- **Motivation:** Has many books collecting dust after reading. Wants to share them with neighbors without losing track.
- **Behavior:** Scans new books with her phone, prints labels, and decides which books are available to borrow. Maintains her digital library regularly.
- **Pain points:** Forgets who she lent what to. Wants hassle-free returns without having to chase people.
- **Usage:** Primarily iOS, occasionally Web to print labels.

### Persona B — Tom, the book seeker
- **Age:** 27 · **Occupation:** Student
- **Motivation:** Reads a lot but prefers not to buy. Wants to borrow books from people nearby instead of purchasing.
- **Behavior:** Searches by title or author. Sends loan requests and waits for approval. Uses OTP for handover when QR is unavailable.
- **Pain points:** Unclear availability, long waits for approval.
- **Usage:** Primarily iOS.

### Persona C — Maria, the active community member
- **Age:** 52 · **Occupation:** Self-employed
- **Motivation:** Organizes a small book club. Donates books she no longer needs — either to specific friends or openly to the community.
- **Behavior:** Creates targeted donation offers for known contacts and open community offers. Uses the messaging feature to coordinate.
- **Pain points:** Too many interested parties, difficult to choose. Doesn't want to forget who she has committed to.
- **Usage:** Web and iOS equally.

---

## 3. Use cases

### UC-01: Import a book and create a copy
**Actor:** Book owner
**Precondition:** Logged in, book physically present
**Flow:**
1. Scan EAN-13 (camera) or enter ISBN manually
2. System fetches metadata (title, author, cover) from OpenLibrary / Google Books
3. User confirms and creates a copy
4. Optional: set `shareable = true` to make the book visible in search
5. Generate and print a QR label (PDF)

### UC-02: Lend a book
**Actors:** Book seeker (requester), book owner (approver)
**Flow:**
1. Tom searches for a book, finds an available copy
2. Tom sends a loan request (`requested`)
3. Lisa receives a notification, approves (`approved`) or rejects
4. Handover proof: both scan the QR code or one enters an OTP (`handover_pending → on_loan`)
5. After the loan period: return using the same flow (`return_pending → returned`)

### UC-03: Return a book
**Actors:** Borrower, owner
**Flow:**
1. Borrower initiates the return, selects method (QR or OTP)
2. Owner confirms the return by scanning or entering an OTP
3. Status changes to `returned`; book becomes available again

### UC-04: Donate a book (targeted)
**Actors:** Book owner (donor), recipient
**Flow:**
1. Lisa selects a copy and creates a targeted offer with a specific recipient (Maria)
2. Maria receives a notification (`donation_invite`), accepts or declines
3. Handover using QR or OTP; ownership transfer

### UC-05: Donate a book (community offer)
**Actors:** Book owner, multiple interested users
**Flow:**
1. Lisa creates a community offer (book visible in search with `for_donation` badge)
2. Interested users register their interest
3. Lisa sees the interest list and selects a recipient
4. Handover as in UC-04; ownership transfers to the recipient

### UC-06: Handover via OTP (fallback)
**Actors:** Both parties
**Flow:**
1. Handover method `otp` selected (QR not available)
2. One party generates a 6-digit code (valid for 10 min)
3. Other party enters the code; system confirms the handover

### UC-07: Re-issue a label
**Actor:** Book owner
**Flow:**
1. User selects a copy → "Re-issue label"
2. QR version is incremented, old label is invalidated
3. New PDF is generated

### UC-08: Archive / restore a copy
**Actor:** Book owner
**Flow:**
1. Set copy to `archived`: disappears from search, no new requests possible
2. Undo: set copy back to `active` (at any time)

### UC-09: Mark as lost / recover
**Actor:** Book owner
**Flow:**
1. After 14 days overdue: owner marks the copy as lost (`status=lost`)
2. If the book turns up: re-activate the copy to put it back into circulation

### UC-10: Send messages
**Actors:** Any two users
**Flow:**
1. Open a thread (from book detail or profile)
2. Send text messages (max. 2,000 characters)
3. Read receipt sent after explicit view; edit/delete within 2 min

---

## 4. Information architecture

```
Community Library
├── Auth
│   ├── Login
│   ├── Register
│   └── Email Verification
│
├── Library  [Tab]
│   ├── My Books (copy list)
│   │   └── Book Detail (own copy)
│   │       ├── Print Label
│   │       ├── Toggle Shareable
│   │       ├── Archive / Activate
│   │       └── Initiate Loan / Donation
│   ├── Add Book
│   │   ├── Scan (camera)
│   │   └── Manual ISBN entry
│   ├── My Loans
│   │   └── Loan Detail
│   │       ├── Handover (start + confirm)
│   │       └── Return (start + confirm)
│   └── My Donations
│       └── Donation Detail
│           ├── Interest list (donor view)
│           └── Select recipient
│
├── Discover  [Tab]
│   ├── Search bar + filters
│   ├── Search results
│   └── Copy Detail (other user's book)
│       └── Send loan request
│
├── Messages  [Tab]
│   ├── Thread list
│   └── Chat
│
├── Notifications  [Tab]
│   └── Notification list
│
└── Profile  [Tab / Menu]
    ├── Profile view
    ├── Settings
    └── Export data / Delete account
```

---

## 5. Screen list

| # | Screen | Platform | Description |
|---|--------|----------|-------------|
| S-01 | Login | Web / iOS | Email + password, link to register |
| S-02 | Register | Web / iOS | Username, email, password |
| S-03 | Email Verification | Web / iOS | Notice + resend link |
| S-04 | My Books | Web / iOS | List of own copies with status badges |
| S-05 | Add Book — Scan | iOS | Camera overlay for EAN-13 |
| S-06 | Add Book — ISBN Entry | Web / iOS | Manual entry + metadata preview |
| S-07 | Book Detail (own copy) | Web / iOS | Cover, metadata, action bar (label, shareable, archive) |
| S-08 | Label Output | Web / iOS | PDF preview + download/print |
| S-09 | My Loans | Web / iOS | Tabs: As lender / As borrower; filterable by state |
| S-10 | Loan Detail | Web / iOS | State, timeline, context-sensitive actions |
| S-11 | Handover — Choose method | Web / iOS | QR scan or OTP |
| S-12 | Handover — QR Scan | iOS | Camera overlay; confirmation after successful scan |
| S-13 | Handover — Generate OTP | Web / iOS | 6-digit code, countdown |
| S-14 | Handover — Enter OTP | Web / iOS | 6-field input |
| S-15 | Return — Choose method | Web / iOS | Same as S-11 |
| S-16 | My Donations | Web / iOS | Tabs: As donor / As recipient |
| S-17 | Create Donation | Web / iOS | Type (targeted / community), select recipient |
| S-18 | Donation Detail | Web / iOS | State, actions (accept, decline, cancel) |
| S-19 | Interest List | Web / iOS | List with avatars and timestamps; select recipient |
| S-20 | Search / Discover | Web / iOS | Search bar, filters (for_loan / for_donation / all), sort |
| S-21 | Search Results | Web / iOS | Cards/list with availability badge |
| S-22 | Copy Detail (other user) | Web / iOS | Cover, metadata, owner, action button (request) |
| S-23 | Send Loan Request | Web / iOS | Desired date, confirmation |
| S-24 | Thread List | Web / iOS | Last message, unread badge |
| S-25 | Chat | Web / iOS | Bubbles, input field, edit/delete menu |
| S-26 | Notifications | Web / iOS | Chronological list; tap opens context screen |
| S-27 | Profile | Web / iOS | Avatar, username, stats |
| S-28 | Settings | Web / iOS | Language, notification channels (for future phases) |
| S-29 | Export data / Delete account | Web / iOS | GDPR: download JSON, account deletion confirmation |
| S-30 | Offline Banner | Web / iOS | Persistent banner when offline |

---

## 6. Component list

### Navigation components
| Component | Description |
|-----------|-------------|
| `BottomTabBar` | iOS: 5 tabs (Library, Discover, Messages, Notifications, Profile) |
| `TopNavBar` | Title, back button, optional actions (e.g. print) |
| `OfflineBanner` | Displayed at top while offline; shows pending sync actions |

### Data display
| Component | Props | Description |
|-----------|-------|-------------|
| `BookCard` | `title`, `author`, `coverUrl`, `status`, `availability` | Compact book card for lists |
| `BookCoverImage` | `coverUrl`, `size` | Cover with fallback placeholder |
| `CopyStatusBadge` | `status: active\|archived\|lost` | Color-coded label on a copy |
| `AvailabilityBadge` | `type: for_loan\|for_donation` | Badge in search results |
| `LoanStateBadge` | `state` | Shows current loan state in color |
| `DonationStateBadge` | `state` | Shows current donation state |
| `TimelineEntry` | `type`, `timestamp`, `actor` | Single entry in an event timeline |
| `UserAvatar` | `username`, `size` | Initials fallback when no image |
| `UnreadBadge` | `count` | Red circle for unread messages |

### Action components
| Component | Description |
|-----------|-------------|
| `PrimaryButton` | Main action on a screen |
| `SecondaryButton` | Secondary / destructive action |
| `ActionBar` | Context-sensitive action bar (adapts to loan/donation state) |
| `ConfirmationDialog` | Modal confirmation for irreversible actions |
| `ShareableToggle` | Toggle for loan visibility |

### Input components
| Component | Description |
|-----------|-------------|
| `SearchBar` | Search field with clear button |
| `FilterChips` | Horizontal chips: All / To Borrow / To Donate |
| `ISBNInput` | Text field with format validation for ISBN-13 |
| `ShortCodeInput` | 13-character field, accepts hyphens, auto-uppercase |
| `OTPInputField` | 6 individual digit fields, auto-focus |
| `MessageInput` | Multi-line input field with character counter (max. 2,000) |

### Scan & Label
| Component | Description |
|-----------|-------------|
| `CameraScanner` | Camera overlay for EAN-13 and QR codes |
| `QRCodeDisplay` | Shows signed QR code for handover/return |
| `OTPCountdown` | Displays the 6-digit code with expiry timer (10 min) |
| `LabelPreview` | PDF preview of the Avery label (QR + short code) |

### Feedback components
| Component | Description |
|-----------|-------------|
| `EmptyState` | Illustration + text when a list is empty |
| `ErrorBanner` | Inline error message (RFC 7807) |
| `LoadingSpinner` | Centered, overlay-capable |
| `SuccessToast` | Brief confirmation after an action |
| `NotificationItem` | Single notification with icon, text, timestamp |

### Chat components
| Component | Description |
|-----------|-------------|
| `ChatBubble` | Own / other message; edit/delete menu for own messages |
| `TombstoneBubble` | Deleted message (placeholder text) |
| `ReadReceipt` | Read checkmark below a message |
| `ThreadListItem` | Preview of last message + unread badge |

---

## 7. Data models

Simplified TypeScript interfaces for frontend/design context. Full schema: [`db-schema.md`](./db-schema.md).

```typescript
// Edition
interface Edition {
  isbn13: string;
  title: string;
  authors: string[];
  publisher?: string;
  publishDate?: string;   // ISO date
  pageCount?: number;
  languageCode?: string;
  coverUrl?: string;
}

// Book Copy
interface BookCopy {
  id: string;             // UUID
  ownerUserId: string;
  isbn13: string;
  edition: Edition;       // joined
  status: 'active' | 'archived' | 'lost';
  shareable: boolean;
  createdAt: string;      // ISO timestamp
}

// Loan
interface Loan {
  id: string;
  bookCopy: BookCopy;
  ownerUserId: string;
  borrowerUserId: string;
  state:
    | 'requested' | 'approved'
    | 'handover_pending' | 'on_loan'
    | 'return_pending' | 'returned'
    | 'canceled_rejected' | 'canceled_expired' | 'canceled_user';
  dueAt?: string;
  createdAt: string;
  updatedAt: string;
  events: LoanEvent[];
}

interface LoanEvent {
  id: string;
  type: 'request' | 'approve' | 'reject' | 'cancel'
      | 'handover_start' | 'handover_confirm'
      | 'return_start' | 'return_confirm'
      | 'auto_expire' | 'mark_lost';
  method?: 'qr' | 'otp' | 'system';
  actorUserId?: string;
  createdAt: string;
}

// Donation
interface Donation {
  id: string;
  bookCopy: BookCopy;
  donorUserId: string;
  recipientUserId?: string;
  type: 'targeted' | 'community';
  state:
    | 'offered' | 'accepted'
    | 'handover_pending' | 'completed'
    | 'declined' | 'canceled' | 'expired';
  createdAt: string;
  updatedAt: string;
  interests?: DonationInterest[];  // community only
}

interface DonationInterest {
  id: string;
  userId: string;
  username: string;   // joined
  createdAt: string;
}

// Message
interface Message {
  id: string;
  threadId: string;
  fromUserId: string;
  body: string;
  editedAt?: string;
  deletedForSender: boolean;
  deletedForRecipient: boolean;
  readAt?: string;
  createdAt: string;
}

interface Thread {
  id: string;
  otherUser: UserSummary;
  lastMessage?: Message;
  unreadCount: number;
}

// Notification
interface Notification {
  id: string;
  type:
    | 'loan_requested' | 'loan_approved' | 'loan_declined'
    | 'handover_pending' | 'due_soon' | 'due' | 'overdue'
    | 'return_confirmed'
    | 'donation_invite' | 'donation_interest_received'
    | 'donation_picked' | 'donation_declined' | 'donation_completed';
  payload: Record<string, unknown>;
  readAt?: string;
  createdAt: string;
}

// User summary
interface UserSummary {
  userId: string;
  username: string;
  displayName?: string;
}
```

---

## 8. API drafts

Full endpoint table: `api-endpoints.md`. The examples below show representative request/response pairs.

### Import a book
```http
POST /editions/import
Idempotency-Key: <uuid>
Content-Type: application/json

{ "isbn13": "9783551551672" }
```
```json
{
  "isbn13": "9783551551672",
  "title": "Harry Potter und der Stein der Weisen",
  "authors": ["J.K. Rowling"],
  "publisher": "Carlsen",
  "publishDate": "1998-07-01",
  "coverUrl": "https://covers.openlibrary.org/b/isbn/9783551551672-M.jpg"
}
```

### Create a copy
```http
POST /book-copies
Idempotency-Key: <uuid>
Content-Type: application/json

{ "isbn13": "9783551551672" }
```
```json
{
  "id": "a1b2c3d4-...",
  "ownerUserId": "user-sub-uuid",
  "isbn13": "9783551551672",
  "status": "active",
  "shareable": false,
  "createdAt": "2025-11-01T10:00:00Z"
}
```

### Send a loan request
```http
POST /loans/request
Idempotency-Key: <uuid>
Content-Type: application/json

{
  "bookCopyId": "a1b2c3d4-...",
  "dueAt": "2025-12-01T00:00:00Z"
}
```
```json
{
  "id": "loan-uuid",
  "bookCopyId": "a1b2c3d4-...",
  "ownerUserId": "owner-sub",
  "borrowerUserId": "borrower-sub",
  "state": "requested",
  "dueAt": "2025-12-01T00:00:00Z",
  "createdAt": "2025-11-01T10:05:00Z"
}
```

### List loans
```http
GET /loans?role=borrower&state=on_loan&page=1&size=20
```
```json
{
  "items": [
    {
      "id": "loan-uuid",
      "state": "on_loan",
      "dueAt": "2025-12-01T00:00:00Z",
      "bookCopy": {
        "id": "a1b2c3d4-...",
        "edition": {
          "title": "Harry Potter und der Stein der Weisen",
          "coverUrl": "..."
        }
      }
    }
  ],
  "total": 1,
  "page": 1,
  "size": 20
}
```

### Generate an OTP (handover fallback)
```http
POST /loans/{id}/otp/generate
Idempotency-Key: <uuid>
```
```json
{
  "code": "482917",
  "expiresAt": "2025-11-01T10:15:00Z"
}
```

### Search
```http
GET /editions?q=tolkien&availability=for_loan&sort=relevance&page=1&size=20
```
```json
{
  "items": [
    {
      "isbn13": "9783608938463",
      "title": "Der Herr der Ringe",
      "authors": ["J.R.R. Tolkien"],
      "coverUrl": "...",
      "availability": "for_loan",
      "copyCount": 2
    }
  ],
  "total": 1
}
```

### Error format (RFC 7807)
```json
{
  "type": "https://community-library.app/errors/loan-state-conflict",
  "title": "Loan state conflict",
  "status": 409,
  "error_code": "LOAN_STATE_CONFLICT",
  "detail": "Loan is not in an approvable state.",
  "instance": "/loans/loan-uuid/approve"
}
```

---

## 9. JSON schema for Figma prompts

This schema describes individual screens in machine-readable form for AI-assisted design generation (e.g. Figma AI, Galileo, Anima).

### Schema definition

```json
{
  "$schema": "community-library/figma-prompt/v1",
  "screen": "<screen-id>",
  "title": "<display name>",
  "platform": "ios | web | both",
  "layout": "list | detail | form | scan | chat | empty",
  "navigation": {
    "type": "tab | push | modal | sheet",
    "backLabel": "<optional>"
  },
  "header": {
    "title": "<title>",
    "actions": ["<icon-name>"]
  },
  "components": [
    {
      "id": "<component-id>",
      "type": "<ComponentName>",
      "props": {},
      "state": "<default | loading | empty | error>",
      "repeat": "<optional: 'list' | number>"
    }
  ],
  "primaryAction": {
    "label": "<button text>",
    "style": "primary | secondary | destructive",
    "enabled": true
  },
  "designTokens": {
    "colorScheme": "light | dark | both",
    "spacing": "compact | regular | comfortable",
    "corner": "rounded | sharp"
  },
  "states": ["<state-1>", "<state-2>"]
}
```

### Example: S-04 My Books

```json
{
  "$schema": "community-library/figma-prompt/v1",
  "screen": "S-04",
  "title": "My Books",
  "platform": "ios",
  "layout": "list",
  "navigation": {
    "type": "tab"
  },
  "header": {
    "title": "Library",
    "actions": ["plus"]
  },
  "components": [
    {
      "id": "filter-chips",
      "type": "FilterChips",
      "props": {
        "options": ["All", "Available to borrow", "Archived"]
      },
      "state": "default"
    },
    {
      "id": "book-list",
      "type": "BookCard",
      "props": {
        "showStatus": true,
        "showShareableToggle": true
      },
      "state": "default",
      "repeat": "list"
    },
    {
      "id": "empty-state",
      "type": "EmptyState",
      "props": {
        "icon": "books",
        "headline": "No books yet",
        "body": "Scan your first book using its EAN-13 barcode."
      },
      "state": "empty"
    }
  ],
  "primaryAction": {
    "label": "Add Book",
    "style": "primary",
    "enabled": true
  },
  "designTokens": {
    "colorScheme": "both",
    "spacing": "regular",
    "corner": "rounded"
  },
  "states": ["default", "empty", "loading"]
}
```

### Example: S-10 Loan Detail

```json
{
  "$schema": "community-library/figma-prompt/v1",
  "screen": "S-10",
  "title": "Loan Detail",
  "platform": "both",
  "layout": "detail",
  "navigation": {
    "type": "push",
    "backLabel": "Loans"
  },
  "header": {
    "title": "Loan",
    "actions": []
  },
  "components": [
    {
      "id": "book-header",
      "type": "BookCard",
      "props": { "size": "large", "showStatus": false },
      "state": "default"
    },
    {
      "id": "loan-state-badge",
      "type": "LoanStateBadge",
      "props": { "state": "on_loan" },
      "state": "default"
    },
    {
      "id": "due-date",
      "type": "InfoRow",
      "props": { "label": "Due on", "value": "2025-12-01" },
      "state": "default"
    },
    {
      "id": "event-timeline",
      "type": "TimelineEntry",
      "props": { "entries": ["request", "approve", "handover_confirm"] },
      "repeat": "list"
    },
    {
      "id": "action-bar",
      "type": "ActionBar",
      "props": {
        "primaryAction": "Start Return",
        "secondaryAction": "Send Message"
      },
      "state": "default"
    }
  ],
  "primaryAction": {
    "label": "Start Return",
    "style": "primary",
    "enabled": true
  },
  "designTokens": {
    "colorScheme": "both",
    "spacing": "regular",
    "corner": "rounded"
  },
  "states": ["requested", "approved", "handover_pending", "on_loan", "return_pending", "returned", "canceled"]
}
```

### Example: S-13 Generate OTP

```json
{
  "$schema": "community-library/figma-prompt/v1",
  "screen": "S-13",
  "title": "Generate OTP",
  "platform": "both",
  "layout": "form",
  "navigation": {
    "type": "sheet",
    "backLabel": "Cancel"
  },
  "header": {
    "title": "Handover Code",
    "actions": ["xmark"]
  },
  "components": [
    {
      "id": "otp-display",
      "type": "OTPCountdown",
      "props": {
        "digits": 6,
        "validitySeconds": 600
      },
      "state": "active"
    },
    {
      "id": "instruction",
      "type": "BodyText",
      "props": {
        "text": "Share this code with the other person. The code is valid for 10 minutes."
      },
      "state": "default"
    }
  ],
  "primaryAction": {
    "label": "Generate New Code",
    "style": "secondary",
    "enabled": true
  },
  "designTokens": {
    "colorScheme": "both",
    "spacing": "comfortable",
    "corner": "rounded"
  },
  "states": ["active", "expired"]
}
```

### Example: S-20 Search / Discover

```json
{
  "$schema": "community-library/figma-prompt/v1",
  "screen": "S-20",
  "title": "Discover",
  "platform": "both",
  "layout": "list",
  "navigation": {
    "type": "tab"
  },
  "header": {
    "title": "Discover",
    "actions": []
  },
  "components": [
    {
      "id": "search-bar",
      "type": "SearchBar",
      "props": { "placeholder": "Title, author or ISBN..." },
      "state": "default"
    },
    {
      "id": "filter-chips",
      "type": "FilterChips",
      "props": {
        "options": ["All", "To Borrow", "To Donate"],
        "sortOptions": ["Relevance", "Newest"]
      },
      "state": "default"
    },
    {
      "id": "results-list",
      "type": "BookCard",
      "props": { "showAvailabilityBadge": true, "showOwner": true },
      "state": "default",
      "repeat": "list"
    },
    {
      "id": "empty-state",
      "type": "EmptyState",
      "props": {
        "icon": "magnifyingglass",
        "headline": "No results",
        "body": "Try a different search term."
      },
      "state": "empty"
    }
  ],
  "designTokens": {
    "colorScheme": "both",
    "spacing": "regular",
    "corner": "rounded"
  },
  "states": ["idle", "searching", "results", "empty", "error"]
}
```
