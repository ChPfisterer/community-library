# Community Library — UI/UX Specification

Diese Spezifikation richtet sich an Design und Frontend-Entwicklung. Sie ergänzt die technische Gesamtspezifikation (`spec.md`) und beschreibt Produktkontext, Personas, Use Cases, Screens, Komponenten, vereinfachte Datenmodelle und API-Vorentwürfe.

---

## 1. Produktbeschreibung

**Community Library** ist eine lokale Buchteiler-Plattform für Communities. Nutzer können physische Bücher innerhalb ihrer Nachbarschaft oder Gruppe verleihen und verschenken.

**Kernfunktionen**
- Buch per EAN-13-Scan importieren und eigene Exemplare anlegen
- QR-Etiketten mit Kurzcode drucken und an Bücher kleben
- Ausleihprozess mit digitalem Handover-Nachweis (QR-Scan oder OTP)
- Bücher gezielt oder öffentlich an Community-Mitglieder verschenken
- Direktnachrichten zwischen Verleihern und Entleihern
- Volltextsuche nach Titel, Autor, ISBN mit Verfügbarkeitsfilter
- Offline-Modus: Scans und Aktionen werden lokal gespeichert und nach Verbindungswiederherstellung synchronisiert
- In-App-Benachrichtigungen für alle relevanten Ereignisse

**Plattformen:** Web (Angular, Mobile-First) · iOS (SwiftUI)
**Zielmarkt:** Deutschland / EU (DSGVO-konform)
**Nicht im MVP:** Android, E-Mail/Push-Benachrichtigungen, Gruppen-Chats, Bewertungen öffentlich abrufbar, unscharfe Suche

---

## 2. User Personas

### Persona A — Lisa, die Buchbesitzerin
- **Alter:** 34 · **Beruf:** Lehrerin
- **Motivation:** Hat viele Bücher, die nach dem Lesen im Regal verstauben. Möchte sie mit Nachbarn teilen, ohne sie zu verlieren.
- **Verhalten:** Scannt neue Bücher mit dem Handy, druckt Etiketten und entscheidet selbst, welche Bücher ausleihbar sind. Pflegt ihre digitale Bibliothek regelmäßig.
- **Schmerzpunkte:** Vergisst, wem sie was geliehen hat. Möchte unkomplizierte Rückgabe ohne Nachfragen.
- **Nutzung:** Primär iOS, gelegentlich Web zum Drucken von Etiketten.

### Persona B — Tom, der Buchsucher
- **Alter:** 27 · **Beruf:** Student
- **Motivation:** Liest viel, kauft aber ungern. Möchte Bücher aus seinem Umfeld leihen, statt sie zu kaufen.
- **Verhalten:** Sucht nach Titeln oder Autoren. Schickt Ausleih-Anfragen und wartet auf Bestätigung. Nutzt OTP bei der Übergabe, wenn QR nicht funktioniert.
- **Schmerzpunkte:** Unklare Verfügbarkeit, lange Wartezeiten auf Bestätigung.
- **Nutzung:** Primär iOS.

### Persona C — Maria, das aktive Community-Mitglied
- **Alter:** 52 · **Beruf:** Selbstständig
- **Motivation:** Organisiert eine kleine Leserunde. Verschenkt Bücher, die sie nicht mehr braucht, gezielt an Freunde oder offen an die Community.
- **Verhalten:** Erstellt gezielte Spendenangebote für bekannte Kontakte und offene Community-Angebote. Nutzt die Nachrichtenfunktion für Absprachen.
- **Schmerzpunkte:** Zu viele Interessenten, schwierige Auswahl. Möchte nicht vergessen, wem sie was zugesagt hat.
- **Nutzung:** Web und iOS gleichwertig.

---

## 3. Use Cases

### UC-01: Buch importieren und Exemplar anlegen
**Akteur:** Buchbesitzer
**Vorbedingung:** Eingeloggt, Buch physisch vorhanden
**Ablauf:**
1. EAN-13 scannen (Kamera) oder ISBN manuell eingeben
2. System holt Metadaten (Titel, Autor, Cover) von OpenLibrary / Google Books
3. Nutzer bestätigt und legt Exemplar an
4. Optional: `shareable = true` setzen, um Buch in Suche sichtbar zu machen
5. QR-Etikett generieren und drucken (PDF)

### UC-02: Buch verleihen
**Akteur:** Buchsucher (Anfragender), Buchbesitzer (Genehmigender)
**Ablauf:**
1. Tom sucht nach Buch, findet verfügbares Exemplar
2. Tom schickt Ausleih-Anfrage (`requested`)
3. Lisa erhält Benachrichtigung, genehmigt (`approved`) oder lehnt ab
4. Übergabe-Nachweis: beide scannen QR-Code oder einer gibt OTP ein (`handover_pending → on_loan`)
5. Nach Ablauf der Leihfrist: Rückgabe analog (`return_pending → returned`)

### UC-03: Buch zurückgeben
**Akteur:** Entleiher, Besitzer
**Ablauf:**
1. Entleiher initiiert Rückgabe, wählt Methode (QR oder OTP)
2. Besitzer bestätigt Rückgabe per Scan oder OTP-Eingabe
3. Status wechselt zu `returned`; Buch wieder verfügbar

### UC-04: Buch gezielt verschenken
**Akteur:** Buchbesitzer (Schenkender), Empfänger
**Ablauf:**
1. Lisa wählt Exemplar, erstellt gezieltes Angebot mit Empfänger (Maria)
2. Maria erhält Benachrichtigung (`donation_invite`), akzeptiert oder lehnt ab
3. Übergabe mit QR oder OTP; Eigentumsübertragung

### UC-05: Buch offen verschenken (Community-Angebot)
**Akteur:** Buchbesitzer, mehrere Interessenten
**Ablauf:**
1. Lisa erstellt Community-Angebot (Buch in Suche mit `for_donation`-Badge sichtbar)
2. Interessenten melden Interesse an
3. Lisa sieht Interessentenliste, wählt einen Empfänger aus
4. Übergabe wie UC-04; Eigentumsübertragung auf Empfänger

### UC-06: Übergabe per OTP (Fallback)
**Akteur:** Beide Parteien
**Ablauf:**
1. Übergabe-Methode `otp` gewählt (QR nicht verfügbar)
2. Eine Partei generiert 6-stelligen Code (10 Min. gültig)
3. Andere Partei gibt Code ein; System bestätigt Übergabe

### UC-07: Etikett neu ausstellen
**Akteur:** Buchbesitzer
**Ablauf:**
1. Nutzer wählt Exemplar → „Etikett neu ausstellen"
2. QR-Version wird inkrementiert, altes Etikett ungültig
3. Neues PDF generiert

### UC-08: Exemplar archivieren / wiederherstellen
**Akteur:** Buchbesitzer
**Ablauf:**
1. Exemplar auf `archived` setzen: verschwindet aus Suche, keine neuen Anfragen möglich
2. Rückgängig: Exemplar auf `active` zurücksetzen (jederzeit)

### UC-09: Als verloren markieren / wiederfinden
**Akteur:** Buchbesitzer
**Ablauf:**
1. Nach 14 Tagen Überfälligkeit: Besitzer markiert als verloren (`status=lost`)
2. Falls Buch doch auftaucht: Exemplar per „Aktivieren" wieder in Umlauf bringen

### UC-10: Nachrichten senden
**Akteur:** Beliebige zwei Nutzer
**Ablauf:**
1. Thread öffnen (aus Buchdetail oder Profil)
2. Textnachrichten senden (max. 2.000 Zeichen)
3. Gelesen-Bestätigung nach explizitem Aufruf; Bearbeiten/Löschen innerhalb 2 Min.

---

## 4. Informationsarchitektur

```
Community Library
├── Auth
│   ├── Login
│   ├── Registrierung
│   └── E-Mail-Verifizierung
│
├── Bibliothek  [Tab]
│   ├── Meine Bücher (Exemplarliste)
│   │   └── Buchdetail (eigenes Exemplar)
│   │       ├── Etikett drucken
│   │       ├── Shareable ein/aus
│   │       ├── Archivieren / Aktivieren
│   │       └── Verleihen / Spenden initiieren
│   ├── Buch hinzufügen
│   │   ├── Scan (Kamera)
│   │   └── Manuelle ISBN-Eingabe
│   ├── Meine Ausleihen
│   │   └── Ausleih-Detail
│   │       ├── Übergabe (Start + Bestätigung)
│   │       └── Rückgabe (Start + Bestätigung)
│   └── Meine Spenden
│       └── Spende-Detail
│           ├── Interessentenliste (Donor-Ansicht)
│           └── Empfänger auswählen
│
├── Entdecken  [Tab]
│   ├── Suchmaske + Filter
│   ├── Suchergebnisse
│   └── Ausgabe-Detail (fremdes Buch)
│       └── Ausleih-Anfrage stellen
│
├── Nachrichten  [Tab]
│   ├── Thread-Liste
│   └── Chat
│
├── Benachrichtigungen  [Tab]
│   └── Benachrichtigungsliste
│
└── Profil  [Tab / Menü]
    ├── Profilansicht
    ├── Einstellungen
    └── Daten exportieren / Konto löschen
```

---

## 5. Screen-Liste

| # | Screen | Plattform | Beschreibung |
|---|--------|-----------|--------------|
| S-01 | Login | Web / iOS | E-Mail + Passwort, Link zur Registrierung |
| S-02 | Registrierung | Web / iOS | Username, E-Mail, Passwort |
| S-03 | E-Mail-Verifizierung | Web / iOS | Hinweis + Resend-Link |
| S-04 | Meine Bücher | Web / iOS | Liste eigener Exemplare mit Status-Badges |
| S-05 | Buch hinzufügen — Scan | iOS | Kamera-Overlay für EAN-13 |
| S-06 | Buch hinzufügen — ISBN-Eingabe | Web / iOS | Manuelle Eingabe + Vorschau Metadaten |
| S-07 | Buchdetail (eigenes Exemplar) | Web / iOS | Cover, Metadaten, Aktionsleiste (Etikett, Shareable, Archivieren) |
| S-08 | Etikett-Ausgabe | Web / iOS | PDF-Vorschau + Download/Drucken |
| S-09 | Meine Ausleihen | Web / iOS | Tabs: Als Verleiher / Als Entleiher; nach Status filterbar |
| S-10 | Ausleih-Detail | Web / iOS | Zustand, Timeline, kontextsensitive Aktionen |
| S-11 | Übergabe — Methode wählen | Web / iOS | QR scannen oder OTP |
| S-12 | Übergabe — QR-Scan | iOS | Kamera-Overlay; Bestätigung nach erfolg. Scan |
| S-13 | Übergabe — OTP generieren | Web / iOS | 6-stelliger Code, Countdown |
| S-14 | Übergabe — OTP eingeben | Web / iOS | 6-Felder-Eingabe |
| S-15 | Rückgabe — Methode wählen | Web / iOS | Analog zu S-11 |
| S-16 | Meine Spenden | Web / iOS | Tabs: Als Schenkender / Als Empfänger |
| S-17 | Spende erstellen | Web / iOS | Art (gezielt / Community), Empfänger auswählen |
| S-18 | Spende-Detail | Web / iOS | Zustand, Aktionen (Annehmen, Ablehnen, Abbrechen) |
| S-19 | Interessentenliste | Web / iOS | Liste mit Avataren und Zeitstempeln; Empfänger auswählen |
| S-20 | Suche / Entdecken | Web / iOS | Suchleiste, Filter (for_loan / for_donation / alle), Sortierung |
| S-21 | Suchergebnisse | Web / iOS | Kacheln/Liste mit Verfügbarkeits-Badge |
| S-22 | Ausgabe-Detail (fremd) | Web / iOS | Cover, Metadaten, Besitzer, Aktionsbutton (Anfragen) |
| S-23 | Ausleih-Anfrage stellen | Web / iOS | Wunsch-Datum, Bestätigung |
| S-24 | Thread-Liste | Web / iOS | Letzte Nachricht, Ungelesen-Badge |
| S-25 | Chat | Web / iOS | Blasen, Eingabefeld, Bearbeiten/Löschen-Menü |
| S-26 | Benachrichtigungen | Web / iOS | Chronologische Liste; Tippen öffnet Kontext-Screen |
| S-27 | Profil | Web / iOS | Avatar, Username, Statistiken |
| S-28 | Einstellungen | Web / iOS | Sprache, Benachrichtigungskanäle (für spätere Phasen) |
| S-29 | Daten exportieren / Konto löschen | Web / iOS | DSGVO: Download JSON, Konto-Lösch-Bestätigung |
| S-30 | Offline-Banner | Web / iOS | Persistenter Banner bei fehlender Verbindung |

---

## 6. Komponentenliste

### Navigations-Komponenten
| Komponente | Beschreibung |
|------------|--------------|
| `BottomTabBar` | iOS: 5 Tabs (Bibliothek, Entdecken, Nachrichten, Benachrichtigungen, Profil) |
| `TopNavBar` | Titel, Back-Button, optionale Aktionen (z. B. Drucken) |
| `OfflineBanner` | Oben eingeblendet, solange keine Verbindung; zeigt ausstehende Sync-Aktionen |

### Datendarstellung
| Komponente | Props | Beschreibung |
|------------|-------|--------------|
| `BookCard` | `title`, `author`, `coverUrl`, `status`, `availability` | Kompakte Buchkarte für Listen |
| `BookCoverImage` | `coverUrl`, `size` | Cover mit Fallback-Platzhalter |
| `CopyStatusBadge` | `status: active\|archived\|lost` | Farbiges Label am Exemplar |
| `AvailabilityBadge` | `type: for_loan\|for_donation` | Badge in Suchergebnissen |
| `LoanStateBadge` | `state` | Zeigt aktuellen Leihstatus farbcodiert |
| `DonationStateBadge` | `state` | Zeigt aktuellen Spendenstatus |
| `TimelineEntry` | `type`, `timestamp`, `actor` | Einzeleintrag in Ereignis-Timeline |
| `UserAvatar` | `username`, `size` | Initialen-Fallback wenn kein Bild |
| `UnreadBadge` | `count` | Roter Kreis für ungelesene Nachrichten |

### Aktions-Komponenten
| Komponente | Beschreibung |
|------------|--------------|
| `PrimaryButton` | Hauptaktion auf einem Screen |
| `SecondaryButton` | Sekundäre / destruktive Aktion |
| `ActionBar` | Kontextsensitive Aktionsleiste (passt sich an Loan/Donation-State an) |
| `ConfirmationDialog` | Modale Bestätigung für irreversible Aktionen |
| `ShareableToggle` | Ein/Aus-Schalter für Ausleih-Sichtbarkeit |

### Eingabe-Komponenten
| Komponente | Beschreibung |
|------------|--------------|
| `SearchBar` | Suchfeld mit Leeren-Button |
| `FilterChips` | Horizontale Chips: Alle / Zum Leihen / Zum Verschenken |
| `ISBNInput` | Textfeld mit Format-Validierung für ISBN-13 |
| `ShortCodeInput` | 13-Zeichen-Feld, akzeptiert Bindestriche, auto-uppercase |
| `OTPInputField` | 6 einzelne Ziffernfelder, auto-focus |
| `MessageInput` | Mehrzeiliges Eingabefeld mit Zeichenzähler (max. 2.000) |

### Scan & Label
| Komponente | Beschreibung |
|------------|--------------|
| `CameraScanner` | Kamera-Overlay für EAN-13 und QR-Codes |
| `QRCodeDisplay` | Zeigt signierten QR-Code für Handover/Return |
| `OTPCountdown` | Anzeige des 6-stelligen Codes mit Ablauf-Timer (10 Min.) |
| `LabelPreview` | PDF-Vorschau des Avery-Etiketts (QR + Kurzcode) |

### Feedback-Komponenten
| Komponente | Beschreibung |
|------------|--------------|
| `EmptyState` | Illustration + Text wenn Liste leer ist |
| `ErrorBanner` | Inline-Fehlermeldung (RFC 7807) |
| `LoadingSpinner` | Zentral, overlay-fähig |
| `SuccessToast` | Kurze Bestätigung nach Aktion |
| `NotificationItem` | Einzelne Benachrichtigung mit Icon, Text, Timestamp |

### Chat-Komponenten
| Komponente | Beschreibung |
|------------|--------------|
| `ChatBubble` | Eigene / fremde Nachricht; Edit/Delete-Menü bei Eigenanteil |
| `TombstoneBubble` | Gelöschte Nachricht (Platzhalter-Text) |
| `ReadReceipt` | Gelesen-Häkchen unterhalb der Nachricht |
| `ThreadListItem` | Vorschau des letzten Satzes + Ungelesen-Badge |

---

## 7. Datenmodelle

Vereinfachte TypeScript-Interfaces für Frontend/Design-Kontext. Vollständiges Schema: `docs/db-schema.md`.

```typescript
// Ausgabe (Edition)
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

// Exemplar (Book Copy)
interface BookCopy {
  id: string;             // UUID
  ownerUserId: string;
  isbn13: string;
  edition: Edition;       // joined
  status: 'active' | 'archived' | 'lost';
  shareable: boolean;
  createdAt: string;      // ISO timestamp
}

// Ausleihe (Loan)
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

// Spende (Donation)
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

// Nachricht (Message)
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

// Benachrichtigung (Notification)
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

// Nutzerzusammenfassung
interface UserSummary {
  userId: string;
  username: string;
  displayName?: string;
}
```

---

## 8. API-Vorentwürfe

Vollständige Endpunkttabelle: `docs/api-endpoints.md`. Die folgenden Beispiele zeigen repräsentative Request/Response-Paare.

### Buch importieren
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

### Exemplar anlegen
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

### Ausleih-Anfrage stellen
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

### Ausleih-Liste abrufen
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

### OTP generieren (Übergabe-Fallback)
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

### Suche
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

### Fehlerformat (RFC 7807)
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

## 9. JSON-Struktur für Figma-Prompts

Dieses Schema beschreibt einzelne Screens in maschinenlesbarer Form für KI-gestützte Design-Generierung (z. B. Figma AI, Galileo, Anima).

### Schema-Definition

```json
{
  "$schema": "community-library/figma-prompt/v1",
  "screen": "<screen-id>",
  "title": "<Anzeigename>",
  "platform": "ios | web | both",
  "layout": "list | detail | form | scan | chat | empty",
  "navigation": {
    "type": "tab | push | modal | sheet",
    "backLabel": "<optional>"
  },
  "header": {
    "title": "<Titel>",
    "actions": ["<icon-name>"]
  },
  "components": [
    {
      "id": "<komponenten-id>",
      "type": "<Komponentenname>",
      "props": {},
      "state": "<default | loading | empty | error>",
      "repeat": "<optional: 'list' | Anzahl>"
    }
  ],
  "primaryAction": {
    "label": "<Button-Text>",
    "style": "primary | secondary | destructive",
    "enabled": true
  },
  "designTokens": {
    "colorScheme": "light | dark | both",
    "spacing": "compact | regular | comfortable",
    "corner": "rounded | sharp"
  },
  "states": ["<Zustand-1>", "<Zustand-2>"]
}
```

### Beispiel: S-04 Meine Bücher

```json
{
  "$schema": "community-library/figma-prompt/v1",
  "screen": "S-04",
  "title": "Meine Bücher",
  "platform": "ios",
  "layout": "list",
  "navigation": {
    "type": "tab"
  },
  "header": {
    "title": "Bibliothek",
    "actions": ["plus"]
  },
  "components": [
    {
      "id": "filter-chips",
      "type": "FilterChips",
      "props": {
        "options": ["Alle", "Ausleihbar", "Archiviert"]
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
        "headline": "Noch keine Bücher",
        "body": "Scanne dein erstes Buch per EAN-13-Code."
      },
      "state": "empty"
    }
  ],
  "primaryAction": {
    "label": "Buch hinzufügen",
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

### Beispiel: S-10 Ausleih-Detail

```json
{
  "$schema": "community-library/figma-prompt/v1",
  "screen": "S-10",
  "title": "Ausleih-Detail",
  "platform": "both",
  "layout": "detail",
  "navigation": {
    "type": "push",
    "backLabel": "Ausleihen"
  },
  "header": {
    "title": "Ausleihe",
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
      "props": { "label": "Fällig am", "value": "01.12.2025" },
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
        "primaryAction": "Rückgabe starten",
        "secondaryAction": "Nachricht senden"
      },
      "state": "default"
    }
  ],
  "primaryAction": {
    "label": "Rückgabe starten",
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

### Beispiel: S-13 OTP generieren

```json
{
  "$schema": "community-library/figma-prompt/v1",
  "screen": "S-13",
  "title": "OTP generieren",
  "platform": "both",
  "layout": "form",
  "navigation": {
    "type": "sheet",
    "backLabel": "Abbrechen"
  },
  "header": {
    "title": "Übergabe-Code",
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
        "text": "Gib diesen Code der anderen Person. Der Code ist 10 Minuten gültig."
      },
      "state": "default"
    }
  ],
  "primaryAction": {
    "label": "Neuen Code generieren",
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

### Beispiel: S-20 Suche / Entdecken

```json
{
  "$schema": "community-library/figma-prompt/v1",
  "screen": "S-20",
  "title": "Entdecken",
  "platform": "both",
  "layout": "list",
  "navigation": {
    "type": "tab"
  },
  "header": {
    "title": "Entdecken",
    "actions": []
  },
  "components": [
    {
      "id": "search-bar",
      "type": "SearchBar",
      "props": { "placeholder": "Titel, Autor oder ISBN..." },
      "state": "default"
    },
    {
      "id": "filter-chips",
      "type": "FilterChips",
      "props": {
        "options": ["Alle", "Zum Leihen", "Zu verschenken"],
        "sortOptions": ["Relevanz", "Neueste"]
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
        "headline": "Keine Ergebnisse",
        "body": "Versuche einen anderen Suchbegriff."
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
