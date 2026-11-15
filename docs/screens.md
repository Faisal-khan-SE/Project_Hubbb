# Screens

Produced in task **G1b** (`docs/G1b-screens-content`) from:
- `docs/source/5_Layered_Architecture.pdf` (screen flows, section 5)
- `docs/requirements.md` (FR-IDs)
- `docs/PROMPT_PACK.md` (additional screens mandated by pack)
- `docs/decisions.md` (D1-D38)

**Last updated:** 2026-11-15  
**Reviewed by:** Founder A (required before merge)

---

## How to read this file

| Field | Description |
|---|---|
| **id** | Stable identifier `SCR-xx`. Do not renumber; add new screens at the end of their section. |
| **Role(s)** | Who sees this screen: `Visitor`, `Client`, `Developer`, `Admin`, `All` (authenticated). |
| **Purpose** | One sentence: what the user is trying to accomplish. |
| **Fields** | Every visible input, display value, or file input on the screen. |
| **Actions** | Buttons, links, or form submissions available to the user. |
| **States** | `loading`, `empty`, `error` are mandatory for every screen. Additional states listed where relevant. |
| **FR-IDs** | At least one requirement from `docs/requirements.md` that this screen implements. |
| **Endpoints** | Named backend operations (exact REST paths are assigned by A-tasks; use the name in `contracts/openapi/backend.yaml`). |

> **Privacy rule (D11, FR-CORE-01):** no screen may expose the email, phone, or real name of one side to the other. Developers are always shown by username only.
>
> **Contact-data rule (NFR-PRV-01):** no screen may display another user's contact details.
>
> **Moderation rule (D8, FR-MSG-01):** any user-entered text field that travels to the other party passes through the moderation bot before display. Screens that have such fields must include an `under_review` state.

---

## Table of contents

1. [Authentication & Onboarding](#1-authentication--onboarding)
2. [Client Screens](#2-client-screens)
3. [Developer Screens](#3-developer-screens)
4. [Shared — Messaging](#4-shared--messaging)
5. [Shared — Notifications](#5-shared--notifications)
6. [Shared — Reviews](#6-shared--reviews)
7. [Shared — Disputes & Support](#7-shared--disputes--support)
8. [Shared — Settings & Account](#8-shared--settings--account)
9. [KYC](#9-kyc)
10. [Invoices](#10-invoices)
11. [Admin Screens](#11-admin-screens)
12. [Public / Visitor Screens](#12-public--visitor-screens)
13. [System / Error Screens](#13-system--error-screens)
14. [Open Questions](#14-open-questions)

---

## 1. Authentication & Onboarding

---

### SCR-01 — Sign Up

**Role(s):** Visitor  
**Purpose:** Register a new account as a client or developer, accept legal documents.

**Fields:**
- Full name (text, required)
- Email address (text, required; validated format)
- Phone number (text, optional; collected but not verified — D20)
- Password (password input, required; strength indicator)
- Confirm password (password input, required)
- Role selector: Client / Developer (required)
- Legal document checkboxes — one per document, each linking to the legal page (D27):
  - Accept Terms of Service (version shown, required)
  - Accept Privacy Policy (version shown, required)
  - Accept Refund & Cancellation Policy (version shown, required)
  - For Developer role: Accept Developer Agreement (version shown, required)
- Accepted document versions are stored (`legal_acceptances` table, D27)

**Actions:**
- Submit (creates account; triggers email OTP — D20)
- Navigate to Login
- Open each legal document link (opens SCR-50, SCR-51, SCR-52, SCR-53)

**States:**
- `loading` — form disabled, spinner while submission is in progress
- `error` — inline field errors (email already registered, password too weak, required checkbox unchecked)
- `success` — redirects to SCR-02 (Email OTP Verify)

**FR-IDs:** FR-AUTH-01, FR-LEG-01, FR-CORE-01

**Endpoints:**
- `POST /auth/register`

---

### SCR-02 — Email OTP Verify

**Role(s):** Client, Developer (newly registered; not yet verified)  
**Purpose:** Verify email address by entering the 6-digit OTP sent to the user's inbox (D20).

**Fields:**
- 6 individual digit boxes (OTP input, auto-advance on digit entry)
- Attempts remaining counter (shown after first failed attempt; account locks after 5 wrong attempts)
- Hint text: "Check your spam folder if you haven't received the email."
- Countdown timer (resend disabled for 60 seconds after each send)

**Actions:**
- Submit OTP
- Resend OTP (enabled only after 60-second cooldown; D20 max 5 attempts total)
- Change email address (returns to SCR-01)

**States:**
- `loading` — OTP submit in progress
- `empty` — initial state; awaiting user input
- `error` — wrong OTP code (shows attempts remaining); OTP expired; too many attempts (account temporarily locked)
- `success` — email verified; redirects to login (SCR-03) or directly to dashboard for the role

**FR-IDs:** FR-AUTH-01, FR-EML-01

**Endpoints:**
- `POST /auth/verify-email`
- `POST /auth/resend-otp`

---

### SCR-03 — Login

**Role(s):** Visitor  
**Purpose:** Authenticate with email and password (D20).

**Fields:**
- Email address (text, required)
- Password (password input, required; show/hide toggle)
- "Remember me" checkbox (optional; controls refresh token lifetime)

**Actions:**
- Submit (logs in; redirects to role dashboard)
- Navigate to Forgot Password (SCR-04)
- Navigate to Sign Up (SCR-01)

**States:**
- `loading` — login request in progress
- `error` — invalid credentials; unverified email (prompt to resend OTP); account suspended (shows restrictions message per D16); rate limit hit
- `success` — redirects to Client Dashboard (SCR-10), Developer Dashboard (SCR-19), or Admin Dashboard (SCR-40)

**FR-IDs:** FR-AUTH-02

**Endpoints:**
- `POST /auth/login`

---

### SCR-04 — Forgot Password — Request OTP

**Role(s):** Visitor  
**Purpose:** Initiate password reset by sending an OTP to the account email (D20).

**Fields:**
- Email address (text, required)

**Actions:**
- Send reset OTP
- Navigate back to Login (SCR-03)

**States:**
- `loading` — sending email
- `error` — email not registered; rate limit exceeded
- `success` — OTP sent message; redirects to SCR-05

**FR-IDs:** FR-AUTH-02, FR-EML-01

**Endpoints:**
- `POST /auth/forgot-password`

---

### SCR-05 — Forgot Password — Enter OTP & New Password

**Role(s):** Visitor  
**Purpose:** Complete password reset by verifying the OTP and setting a new password (D20).

**Fields:**
- 6-digit OTP boxes (same as SCR-02)
- New password (password input, required; strength indicator)
- Confirm new password (password input, required)

**Actions:**
- Submit (verifies OTP + sets new password; revokes all existing refresh tokens)
- Resend OTP (60-second cooldown)
- Navigate back to Login

**States:**
- `loading` — submitting reset
- `error` — wrong OTP; OTP expired; passwords do not match; password too weak
- `success` — password reset; redirects to SCR-03

**FR-IDs:** FR-AUTH-02, FR-EML-01

**Endpoints:**
- `POST /auth/reset-password`

---

### SCR-06 — Places Pop-up (One-time Onboarding, Client)

**Role(s):** Client (shown once after first login following email verification — D31)  
**Purpose:** Collect city and project-category interests to personalise the platform experience (FR-AUTH-04, D31).

**Fields:**
- City (text, required; free text — D31)
- Project categories of interest (multi-select from the category list, 1–5 selections — D31)

**Actions:**
- Save and continue (stores city + category ids; redirects to Client Dashboard)
- Skip (allowed; preferences remain empty; accessible via Settings later)

**States:**
- `loading` — saving preferences
- `error` — save failed
- `success` — redirects to SCR-10

**FR-IDs:** FR-AUTH-04

**Endpoints:**
- `POST /clients/places`

---

### SCR-07 — Developer Profile Setup

**Role(s):** Developer (shown after email verification; can be revisited in Settings)  
**Purpose:** Complete the developer profile required before the account can be verified and bid (FR-AUTH-03).

**Fields:**
- Username (text, required; unique; shown to clients instead of real name — D11)
- Headline (text, required; short professional summary)
- Bio (textarea, optional)
- Years of experience (number, required)
- Portfolio URL (URL, optional)
- Skills (multi-select from the platform skills list, required; at least 1)

**Actions:**
- Save profile
- Continue to Payout Account Setup (SCR-08)
- Skip payout setup (allowed; developer cannot bid until a verified payout account exists — FR-PAY-04)

**States:**
- `loading` — saving
- `error` — username already taken; validation errors
- `incomplete` — profile saved as draft; shown before verification is possible
- `success` — profile saved; KYC and payout still required before bidding

**FR-IDs:** FR-AUTH-03, FR-CORE-01

**Endpoints:**
- `PUT /developers/profile`

---

### SCR-08 — Payout Account Setup

**Role(s):** Developer  
**Purpose:** Add a payout account (bank or mobile wallet) which must be verified by admin before bidding (FR-PAY-04).

**Fields:**
- Account type (select: Bank account / JazzCash / Easypaisa)
- Account number (text, required; masked on display — last 4 digits only after save)
- Account title (text, required)

**Actions:**
- Add account (saves encrypted account number)
- Edit existing account (re-enter number)
- View masked existing account

**States:**
- `loading` — saving
- `error` — invalid account format; save failed
- `pending_verification` — account saved; awaiting admin verification (developer cannot bid yet)
- `verified` — account verified; developer eligible to bid (if profile also complete)

> **Privacy note (D11):** the full account number is never returned to the UI; only the last 4 digits are shown.

**FR-IDs:** FR-PAY-04

**Endpoints:**
- `POST /developers/payout-accounts`
- `GET /developers/payout-accounts`

---

## 2. Client Screens

---

### SCR-10 — Client Dashboard

**Role(s):** Client  
**Purpose:** Overview of the client's active projects and quick access to platform actions.

**Fields:**
- Summary cards: active projects count, awaiting payment count, pending approvals count
- Recent activity list (project name, status, next action needed)
- Chatbot panel (collapsed by default; expands on click — FR-BOT-01)

**Actions:**
- Create new project (→ SCR-11)
- View My Projects (→ SCR-13)
- Open chatbot
- View notifications (→ SCR-32)

**States:**
- `loading` — fetching project summaries
- `empty` — no projects yet; call-to-action to create first project
- `error` — failed to load

**FR-IDs:** FR-PRJ-02, FR-BOT-01

**Endpoints:**
- `GET /clients/dashboard`
- `GET /chatbot/chat` (chatbot panel)

---

### SCR-11 — Create / Upload Project

**Role(s):** Client  
**Purpose:** Upload a new project for the AI to estimate, then publish for bids (FR-PRJ-01).

**Fields:**
- Title (text, required; ≤ 150 chars)
- Description (textarea, required; detailed scope)
- Category (select from categories list, required)
- Skills required (multi-select from skills list, required; ≥ 1)
- File attachments (file upload, optional; SRS/design docs; size/type validated; virus-scanned — D30, A13)
- Budget minimum — PKR (number, required; must be ≥ 80% of AI low estimate after estimate arrives — FR-AI-02)
- Budget maximum — PKR (number, required; must be ≤ 130% of AI high estimate — FR-AI-02)
- Deadline in days (number, required; must be ≥ 70% of AI minimum days — FR-AI-03, D32)
- AI estimate display panel (shown once estimate arrives): price band, days band, complexity, features found, model version (FR-AI-05)
- Band validation message (inline error when budget/deadline falls outside the band)

**Actions:**
- Save as draft (no band check; project stays in `draft` status)
- Request estimate (transitions to `estimate_pending`; estimate job runs in background — D9)
- Publish (only after estimate received and budget/deadline inside the band; transitions to `open`)
- Request admin review (if estimate is wrong; FR-AI-04; requires a written reason; transitions to `under_review`)
- Upload attachment (gets presigned URL; registers metadata after upload)
- Delete attachment

**States:**
- `loading` — saving / requesting estimate
- `estimate_pending` — spinner; "AI is analysing your project, usually under 20 seconds." (NFR-PRF-03)
- `estimate_failed` — AI failed 3 times; prompts to request admin band review (FR-AI-04, D19)
- `under_review` — sent to admin; awaiting band decision
- `draft` — saved; can be edited and published later
- `band_error` — budget or deadline outside the allowed band; clear error message with the allowed range shown
- `error` — network or save error

**FR-IDs:** FR-PRJ-01, FR-AI-01, FR-AI-02, FR-AI-03, FR-AI-04, FR-DEL-01

**Endpoints:**
- `POST /projects` (create draft)
- `PUT /projects/{id}` (update)
- `POST /projects/{id}/request-estimate`
- `POST /projects/{id}/publish`
- `POST /projects/{id}/request-review` (admin band review)
- `POST /projects/{id}/attachments/upload-url`
- `POST /projects/{id}/attachments`
- `DELETE /projects/{id}/attachments/{attachmentId}`
- `GET /projects/{id}/estimate`

---

### SCR-12 — Project Detail (Client View)

**Role(s):** Client (project owner)  
**Purpose:** View full project details, current status, and take status-appropriate actions.

**Fields:**
- Project title, description, category, skills, deadline (days), status badge
- AI estimate panel (price band, days band, complexity — FR-AI-05)
- Budget range set by client
- Agreed price (once a bid is accepted)
- Assigned developer username (once in progress; no contact details — D11)
- Stage progress summary (locked / in_progress / submitted / approved)
- Attachment list (download links for client's own attachments)
- Escrow status banner (after developer pre-start cancellation: indicates escrow and fee are held for the next developer or available for a full refund on request — FR-CAN-05, D35)

**Actions:**
- Edit (only in `draft` status)
- Cancel project (FR-CAN-01..03; confirms with offence-count warning)
- Request escrow refund (available if developer cancelled before starting; refunds held escrow and client fee — FR-CAN-05, D35)
- View bids (→ SCR-15; available when status is `open` or `awaiting_payment`)
- Go to payment (→ SCR-16; when status is `awaiting_payment`)
- View milestones (→ SCR-17; when status is `in_progress`)
- Raise dispute (→ SCR-33; when stage is active)

**States:**
- `loading`
- `draft` — edit available
- `estimate_pending` — estimating indicator
- `open` — bids available
- `awaiting_payment` — 48-hour payment countdown (FR-PAY-01); "Complete payment to start work"
- `in_progress` — stage progress shown
- `developer_cancelled_pre_start` — project returned to `open`; escrow held banner with refund button shown (FR-CAN-05, D35)
- `under_review` — sent to admin for band review
- `completed` — link to leave review (SCR-31)
- `cancelled` — reason and refund status shown
- `disputed` — frozen indicator
- `error`

**FR-IDs:** FR-PRJ-01, FR-PRJ-02, FR-AI-04, FR-AI-05, FR-BID-05, FR-PAY-01, FR-CAN-01, FR-CAN-02, FR-CAN-03, FR-CAN-05, FR-DSP-01, FR-CORE-01

**Endpoints:**
- `GET /projects/{id}`
- `DELETE /projects/{id}/cancel`
- `POST /projects/{id}/escrow/refund-request`

---

### SCR-13 — My Projects (Client)

**Role(s):** Client  
**Purpose:** List all of the client's projects with status and next required action (FR-PRJ-02).

**Fields:**
- Project card per project: title, status badge, category, budget range, next action label
- Filter by status (dropdown)
- Sort by: newest, deadline

**Actions:**
- Open project detail (→ SCR-12)
- Create new project (→ SCR-11)

**States:**
- `loading`
- `empty` — no projects; call-to-action to create first
- `error`

**FR-IDs:** FR-PRJ-02

**Endpoints:**
- `GET /clients/projects`

---

### SCR-14 — Stage Percentage Adjustment

**Role(s):** Client, Developer  
**Purpose:** Adjust stage payment percentages before work starts; must total 100% (FR-DEL-01).

**Fields:**
- Three stage rows: SRS, Working Demo, Source Code
- Percentage input per stage (number; default 20 / 40 / 40)
- Running total indicator (must equal 100%)

**Actions:**
- Save percentages (locked once Stage 1 becomes `in_progress`)
- Reset to defaults

**States:**
- `loading`
- `editable` — before Stage 1 starts
- `locked` — once Stage 1 is active; inputs disabled with explanation
- `error` — total ≠ 100%; negative or zero percentage entered

**FR-IDs:** FR-DEL-01

**Endpoints:**
- `PUT /projects/{id}/milestones/percentages`
- `GET /projects/{id}/milestones`

---

### SCR-15 — Bids View (Client)

**Role(s):** Client  
**Purpose:** Compare all bids side by side; short-list, reject, counter-offer, or accept a bid (FR-BID-06).

**Fields (per bid card):**
- Developer username (never email or real name — D11)
- Developer reliability score (integer 0-100 — D15)
- Developer average rating (1-5 stars)
- Developer completed projects count
- Bid amount (PKR)
- Proposed days
- Proposal text (up to 200 chars; shown after moderation — D8)
- Current round number (1-3) and round history
- Offer expiry countdown (48h per offer — BR-07)
- Bid status badge: `pending`, `shortlisted`, `negotiating`, `accepted`, `rejected`, `expired`

**Actions (per bid):**
- Shortlist / remove from shortlist
- Reject bid (removes from active consideration)
- Counter-offer (opens inline panel — SCR-15a)
- Accept bid (transitions project to `awaiting_payment` — FR-BID-05, D4)

**States:**
- `loading`
- `empty` — no bids yet; "Developers will see your project once it is published."
- `no_bids_active` — all bids rejected or expired
- `awaiting_payment` — one bid accepted; payment prompt shown; other bids still visible but cannot be accepted
- `error`

**FR-IDs:** FR-BID-02, FR-BID-04, FR-BID-05, FR-BID-06, FR-CORE-01, FR-REL-01

**Endpoints:**
- `GET /projects/{id}/bids`
- `POST /projects/{id}/bids/{bidId}/shortlist`
- `POST /projects/{id}/bids/{bidId}/reject`
- `POST /projects/{id}/bids/{bidId}/accept`

---

### SCR-15a — Counter-Offer Panel (inline, Client)

**Role(s):** Client  
**Purpose:** Send a structured counter-offer to the developer (FR-BID-02).

**Fields:**
- Proposed price (PKR, required; must be inside allowed band — FR-BID-03)
- Proposed days (required; must meet 70% rule — FR-AI-03)
- Note (text, optional; ≤ 200 chars; passes moderation — D8)
- Round counter display: "Round X of 3"

**Actions:**
- Send counter-offer (creates a new `bid_offer` row; starts 48h expiry — BR-07)
- Cancel (closes panel)

**States:**
- `loading`
- `band_error` — price or days outside the allowed range
- `round_limit_reached` — 3rd counter already used; this action is disabled (FR-BID-02)
- `under_review` — note is being moderated
- `error`

**FR-IDs:** FR-BID-02, FR-BID-03, FR-MOD-01, FR-MOD-02, FR-MOD-04

**Endpoints:**
- `POST /projects/{id}/bids/{bidId}/offers`

---

### SCR-16 — Payment

**Role(s):** Client  
**Purpose:** Pay the agreed price plus client fee into escrow within the 48-hour window (FR-PAY-01).

**Fields:**
- Agreed price (read-only, PKR)
- Client service fee (read-only; 5% of agreed price, minimum PKR 300 — BR-01)
- Total payable (read-only: agreed price + fee)
- 48-hour payment window countdown timer (D4)
- Payment method selector (JazzCash / Easypaisa / Card / Bank Transfer — FR-PAY-02)
- Payment method-specific fields (rendered after selection; exact fields depend on gateway)

**Actions:**
- Pay now (initiates checkout; calls payment gateway)
- Cancel payment (project returns to `open`; accepted bid back to `pending` — D4)

**States:**
- `loading` — initiating checkout
- `awaiting_gateway` — redirected to gateway; waiting for callback
- `payment_success` — project moves to `in_progress`; congratulations message; link to Milestones
- `payment_failed` — error from gateway; allow retry
- `window_expired` — 48h passed; project already returned to `open`; payment option gone
- `error`

**FR-IDs:** FR-PAY-01, FR-PAY-02, FR-PAY-03, FR-PAY-07, FR-PAY-08, FR-BID-05

**Endpoints:**
- `GET /projects/{id}/payment`
- `POST /projects/{id}/payment/checkout`
- `GET /projects/{id}/payment/status`

---

### SCR-17 — Milestones / Stage Progress (Client View)

**Role(s):** Client  
**Purpose:** Track and act on each delivery stage — approve or request revision (FR-DEL-04).

**Fields (per stage):**
- Stage name (SRS / Working Demo / Source Code)
- Payment percentage and PKR amount
- Status badge: `locked`, `in_progress`, `submitted`, `revision_requested`, `approved`, `disputed`
- Due date (Asia/Karachi — D12)
- Late indicator (if past due; days late shown; penalty accruing — D13)
- Auto-approve countdown (7 calendar days after submission — D14; shown only on `submitted` stages)
- Deliverable preview: document link / demo URL / video link (not source code until approved — FR-DEL-06)
- Version history (each resubmission listed)
- Developer's note (shown after moderation — D8)
- Abandoned unapproved stage decision panel (if developer abandons after submission: prompts client decision within 3 calendar days to approve and pay, or reject and refund — FR-ABN-01)

**Actions:**
- Approve stage (transitions stage to `approved`; triggers payout scheduling — FR-DEL-04)
- Request revision (requires written feedback; re-opens stage for developer — FR-DEL-04)
- Decide on abandoned stage: Approve & Pay / Reject & Refund (available within 3 calendar days of developer abandonment for a submitted unapproved stage — FR-ABN-01)
- End assignment (shown only from day 8 of lateness — FR-LATE-02, D2; confirms abandonment)
- Raise dispute (→ SCR-33)
- Adjust stage percentages (only before Stage 1 starts → SCR-14)

**States:**
- `loading`
- `locked` — stage not yet active
- `in_progress` — developer working
- `submitted` — awaiting client action or auto-approve
- `auto_approve_warning` — within 7-day window; auto-approve date shown prominently
- `abandonment_pending_decision` — developer abandoned with a submitted unapproved stage; 3-day countdown displayed for client decision (FR-ABN-01)
- `revision_requested` — feedback sent; awaiting developer resubmission
- `approved` — payout scheduled
- `disputed` — stage frozen
- `error`

**FR-IDs:** FR-DEL-01, FR-DEL-02, FR-DEL-04, FR-DEL-05, FR-DEL-06, FR-LATE-01, FR-LATE-02, FR-LATE-03, FR-ABN-01, FR-DSP-01

**Endpoints:**
- `GET /projects/{id}/milestones`
- `POST /projects/{id}/milestones/{milestoneId}/approve`
- `POST /projects/{id}/milestones/{milestoneId}/request-revision`
- `POST /projects/{id}/milestones/{milestoneId}/abandonment-decision`
- `POST /projects/{id}/assignments/{assignmentId}/end`

---

### SCR-18 — Payment History & Invoices (Client)

**Role(s):** Client  
**Purpose:** View all payments made and download invoice PDFs (FR-INV-01, D24).

**Fields:**
- Payment history table: date, description (project name + stage), amount, type (deposit / refund / penalty_refund), status
- Invoice list: invoice number (INV-YYYY-xxxxxx series), type, date, amount, download button
- Late penalty refund rows (shown when applicable — D13)

**Actions:**
- Download invoice PDF (owner-only presigned URL — D24)
- Filter by project, date range

**States:**
- `loading`
- `empty` — no payments yet
- `error`

**FR-IDs:** FR-INV-01, FR-PAY-06, FR-PAY-07

**Endpoints:**
- `GET /clients/invoices`
- `GET /invoices/{id}/download`
- `GET /clients/payments`

---

## 3. Developer Screens

---

### SCR-19 — Developer Dashboard

**Role(s):** Developer  
**Purpose:** Overview of the developer's active project assignments, pending bids, earnings summary, reliability score, and chatbot assistance (PDF 2.2 #4, FR-BOT-01, FR-REL-01).

**Fields:**
- Active projects summary / assigned project cards (project title, active stage name, due date, status badge)
- Pending bids summary (project title, bid amount, current round/status, expiry countdown)
- Earnings summary (total earned to date, pending scheduled payouts)
- Reliability score display (0–100 integer, reliability score badge — D15, FR-REL-01)
- Chatbot panel (collapsed by default; expands on click — FR-BOT-01)
- Quick links (Browse Projects, My Bids, Earnings)

**Actions:**
- Open assigned project / stage delivery (→ SCR-24)
- View My Bids (→ SCR-23)
- View Earnings (→ SCR-27)
- Browse Projects (→ SCR-20)
- Open chatbot (FR-BOT-01)
- View notifications (→ SCR-31)

**States:**
- `loading` — fetching dashboard data
- `empty` — new developer with no active projects or bids; prompt to browse open projects
- `account_restricted` — warning banner if account is `under_review` or `suspended` explaining restrictions (D16, FR-ACC-01)
- `error` — failed to load

**FR-IDs:** FR-BOT-01, FR-REL-01, FR-DEL-02, FR-BID-01, FR-PAY-05, FR-ACC-01

**Endpoints:**
- `GET /developers/dashboard`
- `POST /chatbot/chat` (dashboard chatbot proxy)

---

### SCR-20 — Browse Projects

**Role(s):** Developer (must be verified — FR-AUTH-03)  
**Purpose:** Discover open projects that match the developer's skills (FR-AI-05).

**Fields:**
- Search bar (full-text search on title and description)
- Filters: category, skills (multi-select), budget range (min/max PKR), deadline range (days)
- Sort: newest, budget high-to-low, deadline soonest
- Project card per result: title, category, skills, budget range, AI-estimated price band (FR-AI-05), deadline (days), client display name, posted date
- Pagination controls

**Actions:**
- Open project detail (→ SCR-21)
- Apply filter / search

**States:**
- `loading`
- `empty` — no results for the applied filters
- `error`
- `not_eligible` — shown if developer's account is not active or payout account not verified; explains what is needed before bidding

**FR-IDs:** FR-AI-05, FR-CORE-01, FR-CORE-03

**Endpoints:**
- `GET /projects` (with query params: category, skills, budget_min, budget_max, deadline_min, deadline_max, search, page, sort)

---

### SCR-21 — Project Detail (Developer View)

**Role(s):** Developer  
**Purpose:** Read a project's full specification and decide whether to bid (FR-BID-01).

**Fields:**
- Title, description, category, skills, deadline (days)
- AI estimate panel: price band (PKR), days band, complexity, features found (FR-AI-05)
- Allowed bid range (80%–130% of AI estimate — FR-AI-02)
- Client display name (never email or phone — D11)
- Existing bid status (if already bid: status, current offer)
- Stage split (SRS 20% / Demo 40% / Source 40% — BR-06)
- Attachments (text-extracted documents accessible to bidding developers; source zips virus-scanned only — D30)

**Actions:**
- Place bid (opens SCR-22; only if eligible — FR-BID-01)
- Cancel assignment before start (available within 24h of acceptance and before submitting anything; 1st offence warning, 2nd+ strike and -10 reliability drop — FR-CAN-04)
- Open messaging for this project (→ SCR-30; available after first bid is placed)

**States:**
- `loading`
- `open` — bid button active
- `already_bid` — shows current bid status; "Edit bid" or "Withdraw bid" options
- `bid_accepted_other` — developer's bid was not accepted; read-only
- `assigned_pre_start` — project accepted/in-progress; within 24h window and no submissions yet; cancel assignment available with warning (FR-CAN-04)
- `in_progress` — project assigned to this developer; "Go to delivery" link
- `not_eligible` — developer cannot bid (no payout account, under_review, suspended, KYC not approved)
- `error`

**FR-IDs:** FR-BID-01, FR-AI-05, FR-CORE-01, FR-PAY-04, FR-AUTH-03, FR-CAN-04

**Endpoints:**
- `GET /projects/{id}`
- `GET /projects/{id}/my-bid`
- `POST /projects/{id}/assignments/cancel-pre-start`

---

### SCR-22 — Place Bid

**Role(s):** Developer  
**Purpose:** Submit a bid on an open project (FR-BID-01).

**Fields:**
- Bid amount (PKR, required; must be inside the allowed band — FR-BID-01, FR-BID-03)
- Proposed days (required; must meet 70% rule — FR-AI-03)
- Proposal / cover note (textarea, required; passes moderation — D8; max characters not specified in requirements; listed under Open Questions)
- Allowed range reminder (displayed: "Your bid must be between PKR X and PKR Y, and your deadline must be at least N days")

**Actions:**
- Submit bid
- Save draft (optional — not specified; listed under Open Questions)

**States:**
- `loading`
- `band_error` — amount or days outside the band; inline message with the allowed range
- `duplicate_bid` — already bid on this project (one bid per project per developer — FR-BID-01)
- `under_review` — proposal being moderated
- `success` — bid submitted; redirects to My Bids (SCR-23)
- `error`

**FR-IDs:** FR-BID-01, FR-BID-03, FR-MOD-01, FR-MOD-02, FR-MOD-04

**Endpoints:**
- `POST /projects/{id}/bids`

---

### SCR-23 — My Bids (Developer)

**Role(s):** Developer  
**Purpose:** Track all bids the developer has placed (FR-BID-01).

**Fields:**
- Bid card per project: project title, bid amount, proposed days, status badge, current round (if negotiating), expiry countdown
- Status values: `pending`, `shortlisted`, `negotiating`, `accepted`, `rejected`, `withdrawn`, `expired`

**Actions:**
- View project detail (→ SCR-21)
- Respond to counter-offer (→ SCR-23a; when status is `negotiating` and it is the developer's turn)
- Withdraw bid (only in `pending` or `shortlisted` status)
- Cancel assignment before start (when status is `accepted`, within 24h, before first submission — FR-CAN-04)

**States:**
- `loading`
- `empty` — no bids placed yet
- `error`

**FR-IDs:** FR-BID-01, FR-BID-02, FR-CAN-04

**Endpoints:**
- `GET /developers/bids`
- `POST /projects/{id}/assignments/cancel-pre-start`

---

### SCR-23a — Respond to Counter-Offer (Developer)

**Role(s):** Developer  
**Purpose:** Accept, counter, or reject the client's latest offer in a negotiation round (FR-BID-02).

**Fields:**
- Current client offer: price (PKR), days
- Round counter: "Round X of 3"
- Offer expiry countdown (48h — BR-07)
- If countering:
  - New proposed price (PKR, inside band — FR-BID-03)
  - New proposed days (inside 70% rule)
  - Note (optional, ≤ 200 chars, moderated — D8)

**Actions:**
- Accept offer (sets `bid.final_amount`; client notified)
- Counter (sends a new offer; only if rounds remain)
- Reject (ends negotiation for this bid)

**States:**
- `loading`
- `expired` — 48h window passed; offer expired; negotiation ended
- `round_limit_reached` — 3 rounds used; can only accept or reject
- `under_review` — note in moderation
- `error`

**FR-IDs:** FR-BID-02, FR-BID-03, FR-MOD-01, FR-MOD-02, FR-MOD-04

**Endpoints:**
- `POST /projects/{id}/bids/{bidId}/offers/{offerId}/respond`

---

### SCR-24 — Stage Delivery Upload (Developer)

**Role(s):** Developer  
**Purpose:** Submit a deliverable for the active stage (FR-DEL-01..FR-DEL-04).

**Fields:**
- Active stage name and payment percentage
- Due date (Asia/Karachi — D12); days remaining / days overdue indicator
- File upload (via presigned URL; virus-scanned — A13; document text moderation-scanned for contact details — D30)
- Preview / demo URL (text; for Stage 2 working demo)
- Note (textarea, optional; moderated — D8)
- Scan status indicator: `pending` / `clean` / `flagged`
- Version history: list of all previous submissions for this stage

**Actions:**
- Submit deliverable (creates a new version row in `deliverables`)
- Request extension (→ SCR-25; only before the due date — FR-LATE-01)

**States:**
- `loading`
- `in_progress` — no submission yet; upload available
- `scan_pending` — file uploaded; virus scan in progress; "File is being scanned, please wait."
- `scan_flagged` — file flagged; delivery held for admin; developer notified
- `submitted` — awaiting client approval or auto-approve (D14)
- `revision_requested` — client sent feedback; re-upload available; feedback displayed
- `locked` — stage not yet active (previous stage not approved)
- `under_review` — note in moderation
- `error`

**FR-IDs:** FR-DEL-01, FR-DEL-02, FR-DEL-03, FR-DEL-04, FR-DEL-05, FR-LATE-01, FR-LATE-03, FR-MOD-01, FR-MOD-02

**Endpoints:**
- `GET /projects/{id}/milestones`
- `POST /projects/{id}/milestones/{milestoneId}/deliverables/upload-url`
- `POST /projects/{id}/milestones/{milestoneId}/deliverables`

---

### SCR-25 — Extension Request (Developer)

**Role(s):** Developer  
**Purpose:** Request a deadline extension for the active stage before it falls due (FR-LATE-01).

**Fields:**
- Current due date (read-only)
- Requested extension (days, 1–7 — FR-LATE-01)
- Reason (textarea, required; moderated — D8)

**Actions:**
- Submit request (client notified; awaits acceptance or rejection)
- Cancel

**States:**
- `loading`
- `success` — awaiting client response
- `already_requested` — a pending request exists; cannot submit another
- `past_due` — cannot request extension after due date has passed
- `under_review` — reason in moderation
- `error`

**FR-IDs:** FR-LATE-01, FR-MOD-01, FR-MOD-02

**Endpoints:**
- `POST /projects/{id}/milestones/{milestoneId}/extension-requests`

---

### SCR-26 — Extension Request Response (Client)

**Role(s):** Client  
**Purpose:** Accept or reject a developer's extension request; acceptance moves the stage due date (FR-LATE-01).

**Fields:**
- Developer username (not real name — D11)
- Requested additional days
- Reason (read-only; shown after moderation)
- Current due date and new due date if accepted

**Actions:**
- Accept (due date moves; no penalty for the extension period — FR-LATE-01)
- Reject (due date unchanged; developer continues with original deadline)

**States:**
- `loading`
- `pending` — awaiting client decision
- `accepted` — due date updated
- `rejected`
- `expired` — request was not answered in time (see Open Questions for expiry rule)
- `error`

**FR-IDs:** FR-LATE-01

**Endpoints:**
- `POST /projects/{id}/milestones/{milestoneId}/extension-requests/{requestId}/respond`

---

### SCR-27 — Earnings (Developer)

**Role(s):** Developer  
**Purpose:** View per-stage earnings with commission and penalty breakdown, and payout status (FR-PAY-05).

**Fields:**
- Earnings table per stage: project title, stage name, stage amount, late penalty deducted (if any), commission deducted, net payout, payout status badge, payout scheduled date, invoice number and download link
- Total earned to date
- Pending payouts section

**Actions:**
- Download invoice (→ SCR-29; owner-only — D24)
- Filter by project, date range

**States:**
- `loading`
- `empty` — no completed stages yet
- `error`

**FR-IDs:** FR-PAY-05, FR-PAY-06, FR-INV-01, FR-LATE-03, FR-REL-01

**Endpoints:**
- `GET /developers/earnings`
- `GET /invoices/{id}/download`

---

### SCR-28 — Developer Public Profile

**Role(s):** Client, Visitor (read-only)  
**Purpose:** Show a developer's public profile to clients comparing bids (FR-BID-06).

**Fields:**
- Username (not real name — D11)
- Headline and bio
- Skills list
- Reliability score (0-100 integer — D15)
- Average rating (1-5 stars)
- Completed projects count
- Portfolio URL (if provided)
- Reviews received (comment text, rating, project category — no client contact data)

**Actions:**
- None (read-only view)

**States:**
- `loading`
- `not_found`
- `error`

**FR-IDs:** FR-BID-06, FR-REL-01, FR-CORE-01

**Endpoints:**
- `GET /developers/{username}/profile`

---

### SCR-29 — Invoices List & Download (Developer)

**Role(s):** Developer  
**Purpose:** Download payout invoices and receipts (D24, FR-INV-01).

**Fields:**
- Invoice list: number (PAY-YYYY-xxxxxx series), type (`developer_payout`), date, project name, net amount, tax lines (even when 0% — D23), total
- Download button per invoice

**Actions:**
- Download invoice PDF (presigned short-lived URL; owner-only — D24)
- Filter by date range, project

**States:**
- `loading`
- `empty` — no invoices yet
- `error`

**FR-IDs:** FR-INV-01, FR-TAX-01, FR-PAY-05

**Endpoints:**
- `GET /developers/invoices`
- `GET /invoices/{id}/download`

---

## 4. Shared — Messaging

---

### SCR-30 — Conversation / Messaging

**Role(s):** Client, Developer  
**Purpose:** One-to-one project-scoped conversation between client and developer (FR-MSG-01). Available from the first bid onward.

**Fields:**
- Message thread (chronological)
- Per message: sender username (not real name — D11), timestamp, body text, delivery status badge:
  - `delivered` — shown to recipient immediately (risk < 0.3)
  - `under_review` — sender sees "Your message is being reviewed." (risk 0.3-0.7 or > 0.7)
  - `rejected` — sender sees the rejection reason from admin
- Message input (textarea)
- Send button

**Actions:**
- Send message (passes moderation pipeline — D8, FR-MOD-01..04)
- Load earlier messages (pagination)

**States:**
- `loading`
- `empty` — no messages yet; "Messages are available from the first bid onward."
- `messaging_locked` — developer account has 2nd strike; messaging locked for 24h (D16, FR-MSG-06)
- `under_review` — most recent message awaiting moderation; sender sees placeholder
- `error`

**FR-IDs:** FR-MSG-01, FR-MSG-04, FR-MSG-05, FR-MSG-06, FR-MOD-01, FR-MOD-02, FR-MOD-04, FR-CORE-01

**Endpoints:**
- `GET /projects/{id}/conversations/{conversationId}/messages`
- `POST /projects/{id}/conversations/{conversationId}/messages`
- WebSocket/SSE subscription: `WS /ws/conversations/{conversationId}`

---

## 5. Shared — Notifications

---

### SCR-31 — Notifications Bell & Full Notifications Page

**Role(s):** All (authenticated)  
**Purpose:** View and manage real-time in-app notifications (FR-NOT-01).

**Fields:**
- Unread count badge on the bell icon
- Notification list: icon by type, short description, timestamp, project name (linked), read/unread state

Notification types triggered by the system:
- Bid received / counter-offer received / bid accepted
- Payment received / payment window expiry warning
- Stage submitted / stage approved / revision requested
- Payout scheduled / payout paid / payout failed
- Penalty applied
- Dispute opened / dispute resolved
- Strike issued / account status changed
- Message approved / rejected (moderation result)

**Actions:**
- Mark individual notification as read
- Mark all as read
- Navigate to the related entity (project, bid, stage, dispute)

**States:**
- `loading`
- `empty` — no notifications yet
- `error`

**FR-IDs:** FR-NOT-01

**Endpoints:**
- `GET /notifications`
- `POST /notifications/{id}/read`
- `POST /notifications/read-all`
- WebSocket/SSE: `WS /ws/notifications`

---

## 6. Shared — Reviews

---

### SCR-32 — Leave Review

**Role(s):** Client (reviews developer), Developer (reviews client)  
**Purpose:** Submit a one-time rating and comment after project completion (FR-REV-01).

**Fields:**
- Reviewer's perspective (read-only: "You are reviewing [username]")
- Star rating (1-5, required)
- Comment (textarea, optional; moderated — D8)

> **Uniqueness rule (D5):** UNIQUE(project_id, reviewer_id, reviewee_id). On a reopened project with two developers, each can be reviewed separately.

**Actions:**
- Submit review (only after project is `completed`; only once per reviewer-reviewee pair per project — D5)

**States:**
- `loading`
- `not_yet_available` — project not yet completed
- `already_submitted` — review already left; shows the submitted review (read-only)
- `under_review` — comment in moderation
- `error`

**FR-IDs:** FR-REV-01, FR-MOD-01, FR-MOD-02

**Endpoints:**
- `POST /projects/{id}/reviews`
- `GET /projects/{id}/reviews/mine`

---

## 7. Shared — Disputes & Support

---

### SCR-33 — Raise Dispute

**Role(s):** Client, Developer  
**Purpose:** Open a dispute on the current active stage; freezes the stage until admin resolves it (FR-DSP-01).

**Fields:**
- Project name and stage name (read-only)
- Reason (textarea, required; describes the issue)
- Evidence file attachment (optional; virus-scanned)

**Actions:**
- Submit dispute (creates dispute record; milestone status → `disputed`; auto-approve paused — D14)
- Cancel

**States:**
- `loading`
- `success` — dispute submitted; milestone frozen indicator shown
- `already_disputed` — an open dispute already exists on this stage
- `error`

**FR-IDs:** FR-DSP-01

**Endpoints:**
- `POST /projects/{id}/disputes`

---

### SCR-34 — Dispute Detail

**Role(s):** Client, Developer  
**Purpose:** Track the status of an open dispute (FR-DSP-01).

**Fields:**
- Stage frozen indicator
- Dispute reason (read-only)
- Timeline of admin actions
- Resolution outcome (once resolved: release / refund / split percentages; payment rows created)

**States:**
- `loading`
- `open` — submitted; awaiting admin
- `under_review` — admin is reviewing
- `resolved` — outcome and amounts shown
- `error`

**FR-IDs:** FR-DSP-01

**Endpoints:**
- `GET /disputes/{id}`

---

### SCR-35 — Support / Contact

**Role(s):** All (authenticated and visitor)  
**Purpose:** Submit a support ticket to admin (FR-SUP-01). For visitors: a rate-limited contact form.

**Fields:**
- Subject (text, required)
- Message (textarea, required)
- Project reference (select from user's projects, optional — for authenticated users)
- Honeypot field (hidden; not visible to real users; filled by bots and filtered server-side — F7 requirement)

**Actions:**
- Submit ticket (creates `support_tickets` row; admin notified)

**States:**
- `loading`
- `success` — ticket submitted; ticket reference number shown
- `rate_limited` — too many submissions; "Please wait before submitting another request."
- `error`

**FR-IDs:** FR-SUP-01

**Endpoints:**
- `POST /support/tickets`

---

### SCR-36 — Support Ticket Detail

**Role(s):** All (authenticated)  
**Purpose:** Read admin replies to a support ticket.

**Fields:**
- Ticket subject and original message
- Thread of replies (admin and user)
- Status badge: open / in_progress / closed

**Actions:**
- Add reply (authenticated user)

**States:**
- `loading`
- `open` — awaiting admin response
- `closed`
- `error`

**FR-IDs:** FR-SUP-01

**Endpoints:**
- `GET /support/tickets/{id}`
- `POST /support/tickets/{id}/replies`

---

## 8. Shared — Settings & Account

---

### SCR-37 — Profile Settings

**Role(s):** All (authenticated)  
**Purpose:** View and update personal profile information.

**Fields (Client):**
- Display name
- City
- Interests / project categories (1–5 — D31)
- Organisation (optional)

**Fields (Developer):**
- Username (not changeable after verification? — see Open Questions)
- Headline, bio, experience, portfolio URL
- Skills

**Actions:**
- Save changes
- Navigate to Change Password (SCR-38)
- Navigate to Notification Preferences (SCR-39)
- Navigate to Payout Account (SCR-08)
- Navigate to KYC status (SCR-45)
- Request account deletion (→ SCR-37a)

**States:**
- `loading`
- `error`
- `saved` — confirmation toast

**FR-IDs:** FR-AUTH-03, FR-AUTH-04

**Endpoints:**
- `GET /users/me`
- `PUT /clients/profile`
- `PUT /developers/profile`

---

### SCR-37a — Account Status Banner

**Role(s):** Client, Developer  
**Purpose:** Show prominently when the user's account is `under_review` or `suspended`, explaining what is restricted (D16).

**Fields (read-only):**
- Status label: `under_review` or `suspended`
- Explanation of restrictions:
  - `under_review`: cannot place bids, post new projects, or start new conversations; ongoing work continues
  - `suspended`: login only; can view data and open support tickets; payouts already earned still processed
- Contact support link

**Actions:**
- Open support ticket (→ SCR-35)

**FR-IDs:** FR-ACC-01

**Endpoints:**
- `GET /users/me` (account_status field)

---

### SCR-38 — Change Password

**Role(s):** All (authenticated)  
**Purpose:** Update account password.

**Fields:**
- Current password (required)
- New password (required; strength indicator)
- Confirm new password (required)

**Actions:**
- Save (revokes all existing refresh tokens except the current session)

**States:**
- `loading`
- `error` — wrong current password; passwords don't match; new password too weak
- `success` — confirmation; redirected to login

**FR-IDs:** FR-AUTH-02

**Endpoints:**
- `POST /auth/change-password`

---

### SCR-39 — Notification Preferences

**Role(s):** All (authenticated)  
**Purpose:** Control which non-security email notifications are sent (FR-NOT-01).

**Fields:**
- Toggle per notification category (bid updates, payment events, stage events, payout events, disputes, reviews, platform announcements)
- Note: OTP and password-reset emails cannot be turned off

**Actions:**
- Save preferences

**States:**
- `loading`
- `error`
- `saved`

**FR-IDs:** FR-NOT-01

**Endpoints:**
- `GET /users/notification-preferences`
- `PUT /users/notification-preferences`

---

### SCR-37b — Delete Account / Data Export

**Role(s):** All (authenticated)  
**Purpose:** Request account closure and data export.

**Fields:**
- Confirmation text input ("DELETE" to confirm)
- Data export button (downloads user's own data as JSON)

**Actions:**
- Export data (synchronous download of own data — FR-ADM-01 closure feature)
- Delete account (soft delete; personal data removed except records needed for payments/law — D16)

**States:**
- `loading`
- `confirm_required` — requires typing DELETE
- `in_progress_projects` — cannot delete while a project is `in_progress` or `disputed` (see Open Questions)
- `success` — account closure initiated; redirect to landing

**FR-IDs:** FR-ACC-01

**Endpoints:**
- `GET /users/data-export`
- `DELETE /users/me`

---

## 9. KYC

---

### SCR-45 — KYC Upload (Developer)

**Role(s):** Developer  
**Purpose:** Submit KYC documents (CNIC + images) required before account verification (D26, FR-KYC-01).

**Fields:**
- CNIC number (text, required; 13-digit format, dashes allowed; format mask applied)
- CNIC front image (file upload, required; image file types)
- CNIC back image (file upload, required)
- Selfie (file upload, optional)
- Scan status indicator per file: `pending` / `clean` / `flagged`
- Submission status: `not_submitted` / `submitted` / `approved` / `rejected`
- Rejection reason (shown on rejection)
- Note: "Your KYC documents are visible only to platform admins." (D26)

**Actions:**
- Submit KYC (images uploaded via presigned URL; virus-scanned — A13)
- Resubmit (after rejection)

**States:**
- `loading`
- `not_submitted` — prompt to upload
- `scan_pending` — files being scanned
- `scan_flagged` — a file was flagged; re-upload required
- `submitted` — awaiting admin review
- `approved` — green badge; eligible for verification
- `rejected` — reason displayed; resubmit available
- `error`

**FR-IDs:** FR-KYC-01, FR-AUTH-03

**Endpoints:**
- `POST /developers/kyc/upload-url`
- `POST /developers/kyc`
- `GET /developers/kyc/status`

---

## 10. Invoices

*(shared by Client and Developer; individual list screens are SCR-18 and SCR-29; download is part of those screens)*

---

## 11. Admin Screens

---

### SCR-40 — Admin Dashboard

**Role(s):** Admin  
**Purpose:** System-wide overview of key metrics and queues (FR-ADM-01).

**Fields:**
- Summary cards: total users, active projects, queued messages (moderation), open disputes, failed payouts, revenue this month
- Quick links to each admin queue

**Actions:**
- Navigate to Message Moderation Queue (→ SCR-41)
- Navigate to User Verification Queue (→ SCR-42)
- Navigate to Project Moderation (→ SCR-43)
- Navigate to Payments & Disputes (→ SCR-44)
- Navigate to Manual Payouts Queue (→ SCR-46)
- Navigate to Alerts / System Health (→ SCR-47)
- Navigate to Tax Settings (→ SCR-48)
- Navigate to Reports (→ SCR-49)

**States:**
- `loading`
- `error`

**FR-IDs:** FR-ADM-01, FR-ADM-02

**Endpoints:**
- `GET /admin/dashboard`

---

### SCR-41 — Admin Message Moderation Queue

**Role(s):** Admin  
**Purpose:** Review held messages and decide to approve, edit-then-approve, or reject with reason (FR-MSG-02, FR-MSG-03).

**Fields:**
- Queue list: message excerpt, sender username, project, risk score (0-1), flags (categories), time held, SLA indicator (> 2 hours in business hours = breach — D10, D37)
- Message detail view: original text, risk score, layer-1 flags, AI flags, source (message / bid_proposal / offer_note / review_comment etc.)
- Edit text field (admin may edit before approving)
- Rejection reason field

**Actions:**
- Approve (delivers original text to recipient)
- Edit text then approve (delivers admin-edited version; both original and edited stored — FR-MSG-04)
- Reject with reason (sender sees the reason — FR-MSG-03)
- Apply strike (after confirming contact-sharing attempt — FR-MSG-06)

**States:**
- `loading`
- `empty` — queue clear
- `sla_breach` — an item has been queued > 2h in business hours; alert badge (D37)
- `error`

**FR-IDs:** FR-MSG-02, FR-MSG-03, FR-MSG-04, FR-MSG-06, FR-MOD-04, FR-ADM-02

**Endpoints:**
- `GET /admin/moderation/queue`
- `POST /admin/moderation/{messageId}/approve`
- `POST /admin/moderation/{messageId}/edit-approve`
- `POST /admin/moderation/{messageId}/reject`
- `POST /admin/users/{userId}/strikes`

---

### SCR-42 — Admin User Verification

**Role(s):** Admin  
**Purpose:** Review developers' KYC and profiles, verify or reject, change account status (FR-AUTH-03, FR-ADM-01, D26).

**Fields:**
- User list: username, email (admin-only — D11 applies only to inter-user exposure), role, verification_status, account_status, joined date
- User detail: profile fields, KYC document images (via short-lived presigned URLs — audited, D26), CNIC last 4 digits
- Verification status: `pending` / `verified` / `rejected`
- Account status selector: `active` / `under_review` / `suspended` / `closed`
- Reason field (required for status changes)

**Actions:**
- Approve KYC → verify developer (requires approved KYC — D26)
- Reject KYC with reason (developer notified by email)
- Change account_status (with reason; audited — FR-ADM-02)
- View reliability_events history
- View strikes history
- Create another admin (D21)

**States:**
- `loading`
- `empty` — no pending verifications
- `error`

**FR-IDs:** FR-AUTH-03, FR-KYC-01, FR-ACC-01, FR-ADM-01, FR-ADM-02

**Endpoints:**
- `GET /admin/users`
- `GET /admin/users/{id}`
- `POST /admin/users/{id}/kyc/approve`
- `POST /admin/users/{id}/kyc/reject`
- `PUT /admin/users/{id}/status`
- `GET /admin/users/{id}/reliability-events`
- `GET /admin/users/{id}/strikes`
- `POST /admin/users` (create admin)

---

### SCR-43 — Admin Project Moderation

**Role(s):** Admin  
**Purpose:** Review projects sent for band review; override the AI estimate band; set band manually for `estimate_failed` projects (FR-AI-04).

**Fields:**
- Projects list filtered by status: `under_review` (admin band review), `estimate_failed`
- Project detail: title, description, AI estimate, client's budget, reason for review request
- Band override fields: new price_min, price_max, days_min (manual or AI-corrected)
- Reason for override (required; audited)

**Actions:**
- Override band (stores in `estimate_overrides`; audited; project returns to `draft` for client — D29)
- Set band manually for `estimate_failed` (same form; allows project to proceed — D19)
- Reject band review (project returns to `draft` with original band; client notified)

**States:**
- `loading`
- `empty` — no projects pending review
- `error`

**FR-IDs:** FR-AI-04, FR-ADM-01, FR-ADM-02

**Endpoints:**
- `GET /admin/projects?status=under_review`
- `GET /admin/projects?status=estimate_failed`
- `POST /admin/projects/{id}/override-band`

---

### SCR-44 — Admin Payments & Disputes

**Role(s):** Admin  
**Purpose:** View payment details, resolve disputes (FR-DSP-01, FR-ADM-01).

**Fields:**
- Payment search/filter: by project, user, date range, payment type
- Payment detail: amount, type, status, gateway transaction ID (if applicable)
- Open disputes list: project, stage, parties, raised date, reason
- Dispute resolution panel: radio (Release to developer / Refund to client / Split); split percentage inputs

**Actions:**
- Resolve dispute: Release / Refund / Split (creates payment rows in one transaction — FR-DSP-01)
- Filter payments

**States:**
- `loading`
- `empty`
- `error`

**FR-IDs:** FR-DSP-01, FR-PAY-06, FR-ADM-01, FR-ADM-02

**Endpoints:**
- `GET /admin/payments`
- `GET /admin/disputes`
- `POST /admin/disputes/{id}/resolve`

---

### SCR-46 — Admin Manual Payouts Queue

**Role(s):** Admin  
**Purpose:** Process pending payouts manually — record bank/wallet transfer details (D25, FR-PAY-09).

**Fields:**
- Queue list: developer username, masked account number, account type (bank/JazzCash/Easypaisa), amount (net after commission and tax), due date, days overdue indicator, status badge (`awaiting_manual` / `paid` / `failed`)
- Filter: due today / overdue / all
- Payout detail view:
  - Project and stage name
  - Full account number (requires explicit "Reveal" button click; reveal is audited — D25)
  - Transfer reference number field (required to mark paid)
  - Transfer date field (required)
  - Receipt upload (optional)
  - Failure reason field (for mark-failed action)

**Actions:**
- Reveal full account number (audited)
- Mark paid (with reference + date + optional receipt; payout → `paid`; PayoutPaid event fires; invoice created; developer notified)
- Mark failed (with reason; developer notified; retry possible)

**States:**
- `loading`
- `empty` — no pending payouts
- `overdue_alert` — a payout is > 1 business day past due; highlighted row
- `error`

**FR-IDs:** FR-PAY-09, FR-PAY-05, FR-ADM-01, FR-ADM-02

**Endpoints:**
- `GET /admin/payouts?status=awaiting_manual`
- `GET /admin/payouts/{id}/account` (reveal; audited)
- `POST /admin/payouts/{id}/mark-paid`
- `POST /admin/payouts/{id}/mark-failed`

---

### SCR-47 — Admin Alerts & System Health

**Role(s):** Admin  
**Purpose:** View system alerts and service health indicators (FR-OBS-01, A13).

**Fields — Alerts list:**
- Alert type, severity, message, entity reference, raised at, acknowledged status
- Filter by severity (critical / warning / info), type, acknowledged / unacknowledged

**Fields — System health:**
- Health indicator per service: DB, AI service, SMTP (Gmail), storage (MinIO), ClamAV
- Status: UP / DOWN / DEGRADED
- Last successful run time per scheduled job

**Actions:**
- Acknowledge alert (records acknowledging admin and timestamp)
- Filter alerts

**States:**
- `loading`
- `all_healthy` — green indicators across the board
- `degraded` — one or more services DOWN; highlighted
- `error`

**FR-IDs:** FR-OBS-01, FR-ADM-01, FR-EML-01

**Endpoints:**
- `GET /admin/alerts`
- `POST /admin/alerts/{id}/acknowledge`
- `GET /admin/system/health`

---

### SCR-48 — Admin Tax Settings

**Role(s):** Admin  
**Purpose:** View and update tax rules (D23, FR-TAX-01). Changes affect only new invoices; issued invoices are immutable (D24).

**Fields:**
- Tax settings table: tax_type (sales_tax_on_fees / withholding_on_payouts), rate (%), is_active, effective_from, notes
- Warning banner: "Changes apply only to future invoices. Past invoices are immutable."

**Actions:**
- Edit rate and activate/deactivate (audited — FR-ADM-02)
- Add new tax rule

**States:**
- `loading`
- `all_inactive` — rates all 0% inactive (default until tax advisor confirms — D23)
- `error`

**FR-IDs:** FR-TAX-01, FR-ADM-02

**Endpoints:**
- `GET /admin/tax-settings`
- `PUT /admin/tax-settings/{id}`
- `POST /admin/tax-settings`

---

### SCR-49 — Admin Reports & Settings (Fees, Penalties, AI Rates)

**Role(s):** Admin  
**Purpose:** View platform reports and manage fee/penalty settings (FR-ADM-01, FR-CORE-05).

**Fields — Reports:**
- Revenue by period (chart + table)
- User growth
- Project completion rate
- Payout totals

**Fields — Fee Settings:**
- Current active fee_settings row: client_fee_percent, min_fee_pkr, commission_percent
- History of past rows (immutable; changes create a new row)

**Fields — Penalty Rules:**
- penalty_rules table rows: days_late range, rate_percent, cap_percent
- Reliability score deltas

**Fields — AI Rates (Phase 1: read-only):**
- Read-only view of `ai-service/data/rates.yaml` served by `GET /v1/rates` (D34)
- Note: "AI rate table is managed by PR in Phase 1. Editable UI is a Phase 2 item."

**Actions:**
- Update fee_settings (creates a new row; existing projects unaffected — FR-PAY-03)
- Update penalty_rules (creates a new row)
- Export report as CSV

**States:**
- `loading`
- `error`

**FR-IDs:** FR-ADM-01, FR-ADM-02, FR-CORE-05, FR-AI-01

**Endpoints:**
- `GET /admin/reports`
- `GET /admin/fee-settings`
- `POST /admin/fee-settings`
- `GET /admin/penalty-rules`
- `POST /admin/penalty-rules`
- `GET /v1/rates` (AI service — read-only view)

---

## 12. Public / Visitor Screens

---

### SCR-50 — Landing Page

**Role(s):** Visitor  
**Purpose:** Introduce the platform; explain the value proposition for clients and developers; drive sign-ups (FR-PUB-01).

**Fields:**
- Hero section: headline, sub-headline, CTA buttons (Post a Project / Find Work)
- "How it works" summary (3-step client view, 3-step developer view)
- Escrow and 3-stage protection highlight
- Trust indicators (fees, payout model — numbers fetched from public config endpoint, never hard-coded — A11)
- Call-to-action section

**Actions:**
- Sign up as client (→ SCR-01 with role pre-selected)
- Sign up as developer (→ SCR-01 with role pre-selected)
- Learn more → How It Works (→ SCR-54)
- View Pricing (→ SCR-55)

**States:**
- `loading`
- `error` — public config failed to load; show fallback text without numbers

**FR-IDs:** FR-PUB-01

**Endpoints:**
- `GET /api/v1/public/config` (read fee %, commission %, stage percentages for display)

---

### SCR-51 — Terms of Service

**Role(s):** Visitor  
**Purpose:** Display the versioned Terms of Service (D27, FR-LEG-01).

**Fields:**
- Version number and "Last updated" date
- DRAFT banner (non-prod; until L1 lawyer review)
- Section content (rendered from `docs/legal/terms.md`)

**Actions:**
- Accept (linked from SCR-01 sign-up checkboxes only)
- Navigate to other legal pages (footer links)

**States:**
- `loading`
- `error`

**FR-IDs:** FR-LEG-01

**Endpoints:** None (static/SSG render from `docs/legal/terms.md`)

---

### SCR-52 — Privacy Policy

**Role(s):** Visitor  
**Purpose:** Display the versioned Privacy Policy (D27, FR-LEG-01).

*(Same structure as SCR-51; content from `docs/legal/privacy.md`)*

**FR-IDs:** FR-LEG-01

---

### SCR-53 — Refund & Cancellation Policy

**Role(s):** Visitor  
**Purpose:** Display the versioned Refund & Cancellation Policy (D27, FR-LEG-01).

*(Same structure as SCR-51; content from `docs/legal/refund-policy.md`)*

**FR-IDs:** FR-LEG-01

---

### SCR-54 — Developer Agreement

**Role(s):** Visitor, Developer  
**Purpose:** Display the versioned Developer Agreement (D27, FR-LEG-01).

*(Same structure as SCR-51; content from `docs/legal/developer-agreement.md`)*

**FR-IDs:** FR-LEG-01

---

### SCR-55 — How It Works

**Role(s):** Visitor  
**Purpose:** Explain the client and developer journeys step by step (FR-PUB-01).

**Fields:**
- Client journey steps (from `docs/content/how-it-works.md`)
- Developer journey steps (from `docs/content/how-it-works.md`)
- Stage diagram (SRS 20% / Demo 40% / Source 40% — BR-06; numbers from public config)
- Escrow protection explanation

**Actions:**
- Sign up CTA

**States:**
- `loading`
- `error`

**FR-IDs:** FR-PUB-01

**Endpoints:**
- `GET /api/v1/public/config`

---

### SCR-56 — Pricing

**Role(s):** Visitor  
**Purpose:** Show all fees, commissions, and penalty rules; include the PKR 100,000 worked example (FR-PUB-01).

> **Rule:** All numbers MUST be fetched from `GET /api/v1/public/config` at build/revalidate time. No fee or penalty number may be hard-coded in the frontend (FR-CORE-05, A11).

**Fields:**
- Client fee: X% of agreed price (minimum PKR Y) — from public config
- Developer commission: X% per stage after any late penalty — from public config
- Stage percentages — from public config
- Late penalty table (days 1-3: 2%/day; day 4+: 3%/day; cap 25% — from public config)
- Auto-approve: 7 days — from public config
- Payment window: 48 hours — from public config
- Worked example (PKR 100,000 — computed from the config numbers, not typed in)
- Payment methods accepted (from public config)

**Actions:**
- Sign up CTA

**States:**
- `loading`
- `config_error` — fallback message: "Pricing details are temporarily unavailable. Please check back shortly."

**FR-IDs:** FR-PUB-01, FR-CORE-05, FR-PAY-01, FR-PAY-02

**Endpoints:**
- `GET /api/v1/public/config`

---

### SCR-57 — FAQ

**Role(s):** Visitor  
**Purpose:** Answer common questions about the platform (FR-PUB-01). Content from `docs/content/faq.md`.

**Fields:**
- Accordion Q&A list
- Section grouping: Clients, Developers, Payments, Disputes & Cancellations

**Actions:**
- Expand/collapse question
- Contact Support CTA (→ SCR-35)

**States:**
- `loading`
- `error`

**FR-IDs:** FR-PUB-01, FR-BOT-01

**Endpoints:** None (static; content from `docs/content/faq.md`)

---

### SCR-58 — Help / Contact

**Role(s):** Visitor, All  
**Purpose:** Contact form for visitors; support ticket for logged-in users (FR-SUP-01, FR-PUB-01).

**Fields:**
- Name (text, required for visitors; pre-filled for authenticated users)
- Email (text, required for visitors; pre-filled for authenticated users)
- Subject (text, required)
- Message (textarea, required)
- Honeypot field (hidden; not to be filled by real users; prevents automated spam)
- No CAPTCHA service (honeypot is the bot-prevention mechanism — F7 requirement)

**Actions:**
- Submit (rate-limited server-side)

**States:**
- `loading`
- `success` — confirmation with ticket reference
- `rate_limited`
- `error`

**FR-IDs:** FR-SUP-01, FR-PUB-01

**Endpoints:**
- `POST /support/tickets`

---

## 13. System / Error Screens

---

### SCR-60 — 404 Not Found

**Role(s):** All  
**Purpose:** Inform the user that the requested page does not exist.

**Fields:** Friendly message; request_id shown for support reference.

**Actions:** Return to home / dashboard

**FR-IDs:** *(system screen; no specific FR-ID)*

---

### SCR-61 — 500 Server Error

**Role(s):** All  
**Purpose:** Inform the user that an unexpected server error occurred.

**Fields:** Error message; `request_id` prominently shown for support reference (from error response `request_id` field).

**Actions:** Retry; Return to home; Contact support

**FR-IDs:** *(system screen)*

---

### SCR-62 — Offline / Network Error

**Role(s):** All  
**Purpose:** Inform the user that the connection was lost.

**Fields:** Offline indicator; retry message.

**Actions:** Retry (auto-retries when connection restored)

**FR-IDs:** *(system screen)*

---

### SCR-63 — Global Error Boundary

**Role(s):** All  
**Purpose:** Catch unhandled frontend rendering errors; display a recoverable error state without crashing the app.

**Fields:** Generic error message; `request_id` for support; link to contact support.

**Actions:** Reload page; Contact support

**FR-IDs:** *(system screen; supports FR-SUP-01 indirectly)*

---

## 14. Open Questions

The following points are unclear in the available documentation (`docs/source/5_Layered_Architecture.pdf`, `docs/requirements.md`, and `docs/decisions.md`). These must be resolved before the relevant F-tasks begin.

| # | Question | Affects |
|---|---|---|
| OQ-G1b-01 | The proposal text in SCR-22 (Place Bid) has no documented character limit beyond "passes moderation". The 200-char limit in `BR-07` applies only to negotiation **offer notes**, not the initial bid proposal. Confirm whether the bid proposal has its own character limit. | F3, A4 |
| OQ-G1b-02 | Whether a developer can "Save draft" of a bid before submitting is not stated in the requirements. If draft bids are allowed, the `bids` table needs a `draft` status. | F3, A4 |
| OQ-G1b-03 | Whether a developer username is changeable after verification is not specified. If it can change, all existing bid and message records that display username need to reflect the update. | F2, A2 |
| OQ-G1b-04 | The expiry rule for extension requests (SCR-26) — how long the client has to respond before the request auto-expires — is not specified. | F4, A8 |
| OQ-G1b-05 | Whether a client account with an in-progress project can request account deletion (SCR-37b) is not explicitly covered. | F5, A9 |
| OQ-G1b-06 | The SCR numbering for the Pricing page and FAQ uses IDs 56 and 57; the Landing Page is SCR-50. The Legal pages are SCR-51 to SCR-54. If additional public pages are added, the sequence will need extending. No gap policy is defined. | F7 |
| OQ-G1b-07 | Whether the Admin can see/search all invoices (not just unacknowledged payouts) through the same SCR-46 or a separate screen is not fully specified. Task A11 mentions "admin list/search invoices" — a dedicated admin invoice search screen may be needed. | F8, A11 |

