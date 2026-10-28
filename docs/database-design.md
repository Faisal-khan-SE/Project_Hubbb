# Database design (PostgreSQL 16)

Converted in task G1a from `docs/source/4_Database_Architecture.pdf` (36 tables in 7 modules) and adapted to
PostgreSQL and to `docs/decisions.md` (D1-D27). Tables added by the prompt pack are marked **NEW**. Every
table lists the task (and Flyway migration) that creates it. A task may only create the tables listed for it;
any change to a column here needs both founders' approval.

## Conventions (apply to every table)

| Topic | Rule | Source |
|---|---|---|
| Primary keys | `BIGINT GENERATED ALWAYS AS IDENTITY` (small lookup tables: `INT`). Junction tables use composite PKs. | PDF |
| Money | `NUMERIC(12,2)`, Java `BigDecimal` scale 2 HALF_UP. Never float/double. `CHECK (x >= 0)` unless a column is explicitly signed. | D1 |
| Percentages | `NUMERIC(5,2)` (e.g. 20.00, 5.00). | PDF |
| Time | `TIMESTAMPTZ` stored in UTC. Business dates (`due_date`, `scheduled_for`, holidays) are `DATE` in Asia/Karachi. | D12 |
| Enums | `TEXT` + `CHECK (col IN (...))` (PDF says MySQL `ENUM`; PostgreSQL uses CHECK). Values listed per column. | Pack A1 |
| Encrypted values | `BYTEA` (AES-256-GCM, random IV per value, done in the application). | PDF, D26 |
| Audit columns | `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`; `updated_at` where rows change. | PDF |
| Foreign keys | `ON DELETE RESTRICT` everywhere. Users are soft-deleted (`users.deleted_at`), never hard-deleted. | PDF 7 |
| Append-only tables | `payments`, `audit_logs`, `reliability_events`, `invoices`, `message_moderations`: never UPDATE money/decision columns; corrections are new rows. | D1, D18, D24 |
| Status changes | Only through the business layer's state machines (`docs/requirements.md` section 12). | PDF 6 |
| Naming | snake_case, plural table names, `<entity>_id` keys, as in the PDF. | PDF |

## Table inventory and owning task

| # | Table | Module | Origin | Created by (migration) |
|---|---|---|---|---|
| 1 | audit_logs | System | PDF | A1 (V1) |
| 2 | shedlock | System | **NEW** (D22) | A1 (V1) |
| 3 | users | Users & Auth | PDF | A2 (V2) |
| 4 | otp_codes | Users & Auth | PDF | A2 (V2) |
| 5 | refresh_tokens | Users & Auth | **NEW** (A2) | A2 (V2) |
| 6 | client_profiles | Users & Auth | PDF | A2 (V2) |
| 7 | developer_profiles | Users & Auth | PDF | A2 (V2) |
| 8 | skills | Users & Auth | PDF | A2 (V2) |
| 9 | developer_skills | Users & Auth | PDF | A2 (V2) |
| 10 | payout_accounts | Delivery & Payments | PDF | A2 (V2) |
| 11 | legal_acceptances | Users & Auth | **NEW** (D27) | A2 (V2) |
| 12 | email_outbox | System | **NEW** (D20) | A2 (V2) |
| 13 | categories | Projects | PDF | A3 (V3) |
| 14 | projects | Projects | PDF | A3 (V3); columns added by A4, A5 |
| 15 | project_skills | Projects | PDF | A3 (V3) |
| 16 | project_attachments | Projects | PDF | A3 (V3) |
| 17 | ai_estimates | Projects | PDF | A3 (V3) |
| 18 | project_status_history | Projects | **NEW** (A3) | A3 (V3) |
| 19 | estimate_overrides | Projects | **NEW** (A3) | A3 (V3) |
| 20 | platform_settings | Rules | **NEW** (G1a proposal) | A3 (V3); keys added by later tasks |
| 21 | bids | Bidding | PDF | A4 (V4) |
| 22 | bid_offers | Bidding | PDF | A4 (V4) |
| 23 | project_assignments | Projects | PDF | A4 (V4) |
| 24 | fee_settings | Rules | PDF | A5 (V5) |
| 25 | payments | Delivery & Payments | PDF | A5 (V5) |
| 26 | wallets | Delivery & Payments | PDF | A5 (V5) |
| 27 | milestone_templates | Delivery & Payments | PDF | A5 (V5) |
| 28 | milestones | Delivery & Payments | PDF | A5 (V5) |
| 29 | gateway_callbacks | Delivery & Payments | **NEW** (A5) | A5 (V5) |
| 30 | deliverables | Delivery & Payments | PDF | A6 (V6) |
| 31 | payouts | Delivery & Payments | PDF | A6 (V6) |
| 32 | public_holidays | Rules | **NEW** (D6) | A6 (V6) |
| 33 | reliability_events | Rules | **NEW** (D15) | A6 (V6) |
| 34 | conversations | Messaging | PDF | A7 (V7) |
| 35 | messages | Messaging | PDF | A7 (V7) |
| 36 | message_moderations | Messaging | PDF | A7 (V7) |
| 37 | user_strikes | Rules | PDF | A7 (V7) |
| 38 | penalty_rules | Rules | PDF | A8 (V8) |
| 39 | penalties | Rules | PDF | A8 (V8) |
| 40 | extension_requests | Rules | PDF | A8 (V8) |
| 41 | cancellations | Rules | PDF | A8 (V8) |
| 42 | notifications | System | PDF | A9 (V9) |
| 43 | notification_preferences | System | **NEW** (A9) | A9 (V9) |
| 44 | disputes | Feedback & Support | PDF | A9 (V9) |
| 45 | reviews | Feedback & Support | PDF | A9 (V9) |
| 46 | support_tickets | Feedback & Support | PDF | A9 (V9) |
| 47 | support_ticket_replies | Feedback & Support | **NEW** (A9 "admin replies") | A9 (V9) |
| 48 | chatbot_logs | System | PDF | A9 (V9) |
| 49 | tax_settings | Rules | **NEW** (D23) | A11 (V10) |
| 50 | invoices | Delivery & Payments | **NEW** (D24) | A11 (V10) |
| 51 | invoice_sequences | Delivery & Payments | **NEW** (D24) | A11 (V10) |
| 52 | manual_payout_records | Delivery & Payments | **NEW** (D25) | A12 (V11) |
| 53 | alerts | System | **NEW** (A13) | A13 (V12) |
| 54 | kyc_documents | Users & Auth | **NEW** (D26) | A14 (V13) |

36 PDF tables + 18 new = 54.

---

## 1. System (A1)

### 1.1 audit_logs (PDF) - A1 / V1
Who did what and when. Every admin and money action (FR-ADM-02).

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| log_id | BIGINT identity | no | PK | |
| actor_id | BIGINT | yes | FK users | NULL = system job (FK added in V2 when users exists) |
| action | VARCHAR(60) | no | | e.g. BID_ACCEPTED |
| entity_type | VARCHAR(40) | no | | e.g. project |
| entity_id | BIGINT | yes | | |
| details | JSONB | yes | | **added (A1)**: before/after values, reason. Never contact data, secrets or full card/account numbers |
| created_at | TIMESTAMPTZ | no | default now() | |

Indexes: (entity_type, entity_id), (actor_id, created_at).

### 1.2 shedlock (NEW, D22) - A1 / V1
Standard ShedLock JDBC table: `name VARCHAR(64) PK, lock_until TIMESTAMPTZ NOT NULL, locked_at TIMESTAMPTZ NOT NULL, locked_by VARCHAR(255) NOT NULL`.

---

## 2. Users & Auth (A2, A14)

### 2.1 users (PDF) - A2 / V2

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| user_id | BIGINT identity | no | PK | |
| full_name | VARCHAR(100) | no | | Private, never shown to the other side (D11) |
| email | VARCHAR(150) | no | UNIQUE (lower(email)) | Private. Login identifier (D20) |
| phone | VARCHAR(20) | yes | UNIQUE when not null | Private, optional, **not verified** (D20; PDF: required + verified) |
| password_hash | VARCHAR(255) | no | | Argon2 / bcrypt |
| role | TEXT | no | CHECK IN (client, developer, admin) | |
| account_status | TEXT | no | CHECK IN (pending_verification, active, under_review, suspended, closed) | **changed** (C7): PDF pending/active/blocked + D16. Default pending_verification until email verified |
| email_verified_at | TIMESTAMPTZ | yes | | **replaces** PDF `is_email_verified` BOOLEAN |
| must_change_password | BOOLEAN | no | default false | **added** (D21 bootstrap admin) |
| last_login_at | TIMESTAMPTZ | yes | | **added** (A2) |
| status_reason | VARCHAR(255) | yes | | **added** (D16): admin reason for under_review/suspended |
| created_at | TIMESTAMPTZ | no | default now() | |
| updated_at | TIMESTAMPTZ | no | default now() | |
| deleted_at | TIMESTAMPTZ | yes | | **added**: soft delete / account closure (PDF 7, NFR-PRV-02) |

Removed from PDF: `is_phone_verified` (D20).

### 2.2 otp_codes (PDF) - A2 / V2

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| otp_id | BIGINT identity | no | PK | |
| user_id | BIGINT | no | FK users | |
| code_hash | VARCHAR(255) | no | | Never the raw code |
| purpose | TEXT | no | CHECK IN (signup, password_reset, login) | PDF values |
| channel | TEXT | no | CHECK IN (email) default 'email' | **added** (D20); room for sms later |
| expires_at | TIMESTAMPTZ | no | | 10 minutes (A2) |
| used_at | TIMESTAMPTZ | yes | | **replaces** PDF `is_used` BOOLEAN |
| attempts | SMALLINT | no | default 0, CHECK 0..5 | **added** (A2): max 5 |
| created_at | TIMESTAMPTZ | no | default now() | **added**: resend cooldown 60 s |

Index: (user_id, purpose, created_at DESC).

### 2.3 refresh_tokens (NEW, A2) - A2 / V2

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| token_id | BIGINT identity | no | PK | |
| user_id | BIGINT | no | FK users | |
| token_hash | VARCHAR(255) | no | UNIQUE | |
| family_id | UUID | no | | Rotation family; reuse revokes the family |
| expires_at | TIMESTAMPTZ | no | | |
| revoked_at | TIMESTAMPTZ | yes | | |
| device_info | VARCHAR(255) | yes | | |
| created_at | TIMESTAMPTZ | no | default now() | |

Index: (user_id, revoked_at).

### 2.4 client_profiles (PDF) - A2 / V2

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| client_id | BIGINT | no | PK, FK users (1:1) | |
| display_name | VARCHAR(60) | no | | Name shown to developers |
| city | VARCHAR(60) | yes | | From the Places pop-up |
| interests | TEXT[] | yes | | **added** (FR-AUTH-04): Places interests; format pending Q4 |
| places_completed_at | TIMESTAMPTZ | yes | | **added**: pop-up shown only once |
| organization | VARCHAR(100) | yes | | |
| avatar_url | VARCHAR(255) | yes | | Storage key |

Removed from PDF: `cancellation_count` (derived, D3 / C8).

### 2.5 developer_profiles (PDF) - A2 / V2

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| developer_id | BIGINT | no | PK, FK users (1:1) | |
| username | VARCHAR(40) | no | UNIQUE (lower(username)) | Public handle |
| headline | VARCHAR(120) | yes | | |
| bio | TEXT | yes | | Moderated for contact info |
| experience_years | SMALLINT | yes | CHECK 0..60 | |
| portfolio_url | VARCHAR(255) | yes | | Checked by admin for contact info |
| verification_status | TEXT | no | CHECK IN (pending, verified, rejected) default pending | Verify requires approved KYC (D26) |
| avg_rating | NUMERIC(2,1) | yes | CHECK 1.0..5.0 | Cached from reviews |
| completed_count | INT | no | default 0 | Cached |
| reliability_score | SMALLINT | no | default 100, CHECK 0..100 | **changed** (D15 / C10): cached from reliability_events |

Removed from PDF: `abandoned_count` (derived, C8).

### 2.6 skills (PDF) - A2 / V2
`skill_id INT identity PK`, `name VARCHAR(50) NOT NULL UNIQUE`. Seed a starter list (React, Node.js, Python, Java, Flutter, PHP, Laravel, Django, .NET, SQL, PostgreSQL, MongoDB, AWS, UI/UX, WordPress, Android, iOS, Machine Learning).

### 2.7 developer_skills (PDF) - A2 / V2
`developer_id BIGINT FK developer_profiles`, `skill_id INT FK skills`, PK (developer_id, skill_id).

### 2.8 payout_accounts (PDF) - A2 / V2

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| account_id | BIGINT identity | no | PK | |
| developer_id | BIGINT | no | FK users | |
| method | TEXT | no | CHECK IN (jazzcash, easypaisa, bank) | |
| account_title | VARCHAR(100) | no | | |
| bank_name | VARCHAR(100) | yes | | **added**: needed for manual transfers (D25) |
| account_number_enc | BYTEA | no | | AES-256-GCM (PAYOUT_ENCRYPTION_KEY) |
| account_last4 | CHAR(4) | no | | **added**: masked display |
| is_default | BOOLEAN | no | default false | One default per developer (partial unique index) |
| is_verified | BOOLEAN | no | default false | Set by admin (test transfer) |
| created_at | TIMESTAMPTZ | no | default now() | |

Index: UNIQUE (developer_id) WHERE is_default.

### 2.9 legal_acceptances (NEW, D27) - A2 / V2
`acceptance_id BIGINT PK`, `user_id FK users NOT NULL`, `document TEXT CHECK IN (terms, privacy, refund_policy, developer_agreement)`, `version VARCHAR(20) NOT NULL`, `accepted_at TIMESTAMPTZ NOT NULL DEFAULT now()`. UNIQUE (user_id, document, version).

### 2.10 email_outbox (NEW, D20) - A2 / V2

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| email_id | BIGINT identity | no | PK | |
| user_id | BIGINT | yes | FK users | |
| to_address | VARCHAR(150) | no | | |
| template | VARCHAR(60) | no | | e.g. verify_email, password_reset |
| payload | JSONB | no | | Template variables. **Never** the raw OTP after sending |
| is_security | BOOLEAN | no | default false | OTP/reset: always sent, cannot be turned off |
| status | TEXT | no | CHECK IN (queued, sent, failed) | |
| attempts | SMALLINT | no | default 0 | |
| last_error | VARCHAR(500) | yes | | |
| created_at | TIMESTAMPTZ | no | default now() | |
| sent_at | TIMESTAMPTZ | yes | | Daily count for EMAIL_DAILY_LIMIT |

Index: (status, created_at), (sent_at).

### 2.11 kyc_documents (NEW, D26) - A14 / V13

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| kyc_id | BIGINT identity | no | PK | |
| user_id | BIGINT | no | FK users | Developer |
| cnic_number_enc | BYTEA | yes | | KYC_ENCRYPTION_KEY; on the cnic_front row |
| cnic_last4 | CHAR(4) | yes | | |
| cnic_hash | VARCHAR(128) | yes | | Keyed hash for duplicate detection |
| doc_type | TEXT | no | CHECK IN (cnic_front, cnic_back, selfie) | |
| storage_key | VARCHAR(255) | yes | | Private bucket; NULL after retention delete |
| scan_status | TEXT | no | CHECK IN (pending, clean, flagged) | |
| status | TEXT | no | CHECK IN (submitted, approved, rejected) | |
| reviewer_id | BIGINT | yes | FK users | |
| reviewed_at | TIMESTAMPTZ | yes | | |
| rejection_reason | VARCHAR(255) | yes | | |
| delete_after | DATE | yes | | Closure + 90 days |
| created_at | TIMESTAMPTZ | no | default now() | |

Index: (cnic_hash), (status).

---

## 3. Projects (A3, A4)

### 3.1 categories (PDF) - A3 / V3
`category_id INT identity PK`, `name VARCHAR(50) NOT NULL UNIQUE`. Seed: Web, Mobile, Desktop, AI/ML, E-commerce, Other.

### 3.2 projects (PDF) - A3 / V3

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| project_id | BIGINT identity | no | PK | |
| client_id | BIGINT | no | FK users | Owner |
| category_id | INT | no | FK categories | |
| title | VARCHAR(150) | no | | |
| description | TEXT | no | | |
| budget_min | NUMERIC(12,2) | yes | CHECK budget_min <= budget_max | NULL while draft |
| budget_max | NUMERIC(12,2) | yes | | |
| deadline | DATE | yes | | PDF |
| deadline_days | INT | yes | CHECK > 0 | **proposed** (Q5): days used for the 70% rule |
| allowed_min | NUMERIC(12,2) | yes | | 80% of AI low (or admin override) |
| allowed_max | NUMERIC(12,2) | yes | | 130% of AI high (or admin override) |
| agreed_amount | NUMERIC(12,2) | yes | | Final price after bid/negotiation |
| client_fee | NUMERIC(12,2) | yes | | Fee snapshot at acceptance (FR-PAY-03) |
| client_fee_percent_snapshot | NUMERIC(5,2) | yes | | **added by A5** (FR-PAY-03) |
| min_client_fee_snapshot | NUMERIC(12,2) | yes | | **added by A5** |
| commission_percent_snapshot | NUMERIC(5,2) | yes | | **added by A5** |
| reopen_count | SMALLINT | no | default 0 | |
| status | TEXT | no | CHECK IN (draft, estimate_pending, estimate_failed, open, awaiting_payment, in_progress, under_review, completed, cancelled, disputed) | D19 / C6 |
| estimate_attempts | SMALLINT | no | default 0 | **added** (A3): 3 failures -> estimate_failed |
| assigned_developer_id | BIGINT | yes | FK users | **added by A4** migration (avoids circular FK) |
| accepted_bid_id | BIGINT | yes | FK bids | **added by A4** migration |
| payment_due_at | TIMESTAMPTZ | yes | | **added by A4**: acceptance + 48 h (D4) |
| search_vector | TSVECTOR | yes | GIN index | **added** (A3): full-text search on title + description |
| created_at | TIMESTAMPTZ | no | default now() | |
| updated_at | TIMESTAMPTZ | no | default now() | |
| published_at | TIMESTAMPTZ | yes | | When status became open |

Constraints: CHECK budget_min <= budget_max; CHECK (status NOT IN ('open') OR (budget_min >= allowed_min AND budget_max <= allowed_max)) (PDF 7: band enforced in DB too, plus in the service). Indexes: (status, category_id), (client_id), GIN (search_vector).

### 3.3 project_skills (PDF) - A3 / V3
`project_id FK projects`, `skill_id FK skills`, PK (project_id, skill_id).

### 3.4 project_attachments (PDF) - A3 / V3

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| attachment_id | BIGINT identity | no | PK | |
| project_id | BIGINT | no | FK projects | |
| file_url | VARCHAR(255) | no | | Storage key, not a public URL |
| file_name | VARCHAR(150) | no | | |
| file_size_kb | INT | no | CHECK > 0 | |
| content_type | VARCHAR(100) | no | | **added** (A3 type validation) |
| scan_status | TEXT | no | CHECK IN (pending, clean, flagged) | **added** (A13: all uploads scanned) |
| extracted_text | TEXT | yes | | **added**: sent to the AI estimate as attachment_texts |
| uploaded_at | TIMESTAMPTZ | no | default now() | |

### 3.5 ai_estimates (PDF) - A3 / V3

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| estimate_id | BIGINT identity | no | PK | |
| project_id | BIGINT | no | FK projects | History kept (reopen = new row) |
| price_min | NUMERIC(12,2) | no | CHECK price_min <= price_max | |
| price_max | NUMERIC(12,2) | no | | |
| days_min | INT | no | CHECK days_min <= days_max | |
| days_max | INT | no | | |
| complexity | TEXT | no | CHECK IN (low, medium, high) | **added** (contract) |
| features | JSONB | no | | **added** (contract) |
| covers_from_stage | SMALLINT | no | default 1 | **added** (FR-AI-05 reopen) |
| model_version | VARCHAR(30) | no | | FR-AI-06 |
| created_at | TIMESTAMPTZ | no | default now() | |

### 3.6 project_status_history (NEW, A3) - A3 / V3
`history_id BIGINT PK`, `project_id FK projects`, `from_status TEXT`, `to_status TEXT NOT NULL`, `changed_by FK users NULL (system)`, `reason VARCHAR(255)`, `created_at TIMESTAMPTZ`.

### 3.7 estimate_overrides (NEW, A3) - A3 / V3
`override_id BIGINT PK`, `project_id FK projects`, `admin_id FK users`, `old_allowed_min, old_allowed_max, new_allowed_min, new_allowed_max NUMERIC(12,2)`, `old_days_min, new_days_min INT`, `client_reason TEXT`, `admin_reason TEXT NOT NULL`, `created_at`. (FR-AI-04)

### 3.8 platform_settings (NEW, G1a proposal) - A3 / V3
Non-money rule numbers that the PDF wants as data ("rules as data") but that do not fit `fee_settings` or `penalty_rules`.

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| setting_key | VARCHAR(60) | no | PK | |
| value_numeric | NUMERIC(12,2) | yes | | |
| value_text | VARCHAR(255) | yes | | |
| description | VARCHAR(255) | no | | |
| updated_by | BIGINT | yes | FK users | Changes audited |
| updated_at | TIMESTAMPTZ | no | default now() | |

Seed keys (task that first needs them adds them): `band_min_percent`=80, `band_max_percent`=130, `deadline_min_percent`=70 (A3); `offer_valid_hours`=48, `max_offer_rounds`=3, `payment_window_hours`=48 (A4); `auto_approve_days`=7 (A6); `strike_lock_hours`=24, `strike_review_count`=3, `strike_expiry_months`=6 (A7); `reliability_late_delta`=-5, `reliability_dev_cancel_delta`=-10, `reliability_abandon_delta`=-20, `reliability_on_time_delta`=2 (A6/A8); `abandonment_late_day`=8, `extension_max_days`=7 (A8).

### 3.9 project_assignments (PDF) - A4 / V4

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| assignment_id | BIGINT identity | no | PK | |
| project_id | BIGINT | no | FK projects | |
| developer_id | BIGINT | no | FK users | |
| bid_id | BIGINT | no | FK bids, UNIQUE | 1:1 with the bid |
| starts_from_stage | SMALLINT | no | default 1 | e.g. 3 when resuming |
| started_at | TIMESTAMPTZ | no | | Created on payment success (D4) |
| ended_at | TIMESTAMPTZ | yes | | |
| end_reason | TEXT | yes | CHECK IN (completed, abandoned, cancelled_by_client, cancelled_by_developer) | |

Index: UNIQUE (project_id) WHERE ended_at IS NULL (one active developer).

---

## 4. Bidding (A4)

### 4.1 bids (PDF) - A4 / V4

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| bid_id | BIGINT identity | no | PK | |
| project_id | BIGINT | no | FK projects | |
| developer_id | BIGINT | no | FK users | |
| amount | NUMERIC(12,2) | no | CHECK > 0 | Inside band (service) |
| proposed_days | INT | no | CHECK > 0 | |
| proposal | TEXT | no | | Moderated |
| covers_from_stage | SMALLINT | no | default 1 | Reopened projects |
| final_amount | NUMERIC(12,2) | yes | | After negotiation |
| final_days | INT | yes | | **added**: days from the accepted offer |
| status | TEXT | no | CHECK IN (pending, negotiating, shortlisted, accepted, rejected, withdrawn) | |
| created_at | TIMESTAMPTZ | no | default now() | |
| updated_at | TIMESTAMPTZ | no | default now() | |

Constraints: UNIQUE (project_id, developer_id). Indexes: (project_id, status), (developer_id).

### 4.2 bid_offers (PDF) - A4 / V4

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| offer_id | BIGINT identity | no | PK | |
| bid_id | BIGINT | no | FK bids | |
| offered_by | BIGINT | no | FK users | |
| round_no | SMALLINT | no | CHECK 1..3 | |
| amount | NUMERIC(12,2) | no | CHECK > 0 | Inside band (service) |
| days | INT | no | CHECK > 0 | |
| note | VARCHAR(200) | yes | | **added** (FR-BID-03): moderated |
| status | TEXT | no | CHECK IN (pending, accepted, countered, rejected, expired) | |
| expires_at | TIMESTAMPTZ | no | | created_at + 48 h |
| created_at | TIMESTAMPTZ | no | default now() | |

Index: (status, expires_at) for the expiry job.

---

## 5. Delivery & Payments (A5, A6, A11, A12)

### 5.1 fee_settings (PDF) - A5 / V5

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| setting_id | INT identity | no | PK | |
| client_fee_percent | NUMERIC(5,2) | no | CHECK 0..100 | Default 5.00 |
| developer_commission_percent | NUMERIC(5,2) | no | CHECK 0..100 | Default 10.00 |
| min_client_fee | NUMERIC(12,2) | no | CHECK >= 0 | Default 300.00 |
| effective_from | TIMESTAMPTZ | no | | Newest row with effective_from <= now() applies to new acceptances |
| created_by | BIGINT | yes | FK users | **added** (audited admin change) |

### 5.2 payments (PDF) - A5 / V5

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| payment_id | BIGINT identity | no | PK | Append-only (D1) |
| project_id | BIGINT | no | FK projects | |
| milestone_id | BIGINT | yes | FK milestones | NULL for full deposit |
| payer_id | BIGINT | yes | FK users | NULL = platform |
| payee_id | BIGINT | yes | FK users | NULL while held / platform |
| amount | NUMERIC(12,2) | no | CHECK > 0 | |
| platform_fee | NUMERIC(12,2) | yes | | Commission (PDF) |
| type | TEXT | no | CHECK IN (deposit, client_fee, release, commission, penalty_refund, compensation, refund, tax_withheld) | `tax_withheld` **added by A11** |
| status | TEXT | no | CHECK IN (pending, held, released, refunded, failed) | |
| gateway | TEXT | yes | CHECK IN (jazzcash, easypaisa, card, bank_transfer, mock) | `bank_transfer` (C11), `mock` (dev) **added** |
| gateway_txn_id | VARCHAR(100) | yes | UNIQUE WHERE NOT NULL | FR-PAY-08 |
| related_payment_id | BIGINT | yes | FK payments | **added**: links a refund/release to its source row |
| created_at | TIMESTAMPTZ | no | default now() | |

Indexes: (project_id), UNIQUE (gateway_txn_id) WHERE gateway_txn_id IS NOT NULL.

### 5.3 wallets (PDF) - A5 / V5
Cached summary; `payments` is the source of truth (D18).

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| wallet_id | BIGINT identity | no | PK | |
| user_id | BIGINT | no | FK users, UNIQUE | 1:1 |
| available_balance | NUMERIC(12,2) | no | CHECK >= 0 | |
| held_balance | NUMERIC(12,2) | no | CHECK >= 0 | |
| updated_at | TIMESTAMPTZ | no | default now() | |

### 5.4 milestone_templates (PDF) - A5 / V5

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| template_id | INT identity | no | PK | |
| stage_type | TEXT | no | CHECK IN (srs, demo, source_code, custom) | |
| sequence_no | SMALLINT | no | | 1, 2, 3 |
| title | VARCHAR(100) | no | | |
| default_percent | NUMERIC(5,2) | no | CHECK > 0 | |
| is_active | BOOLEAN | no | default true | |

Seed: (srs, 1, "SRS document", 20.00), (demo, 2, "Working demo", 40.00), (source_code, 3, "Source code and handover", 40.00). Active rows must sum to 100 (service check).

### 5.5 milestones (PDF) - A5 / V5

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| milestone_id | BIGINT identity | no | PK | |
| project_id | BIGINT | no | FK projects | |
| template_id | INT | yes | FK milestone_templates | NULL for custom |
| stage_type | TEXT | no | CHECK IN (srs, demo, source_code, custom) | |
| sequence_no | SMALLINT | no | | |
| title | VARCHAR(150) | no | | |
| payment_percent | NUMERIC(5,2) | no | CHECK > 0 | Sum per project = 100 (service) |
| amount | NUMERIC(12,2) | no | CHECK >= 0 | agreed x percent / 100, Money.split remainder fix |
| due_date | DATE | yes | | Moves on accepted extension. How it is set: Q6 |
| assignment_id | BIGINT | yes | FK project_assignments | Who did this stage |
| late_days | SMALLINT | no | default 0 | D13 |
| penalty_amount | NUMERIC(12,2) | no | default 0 | Refunded to client |
| status | TEXT | no | CHECK IN (locked, in_progress, submitted, revision_requested, approved, disputed) | `disputed` **added** (A6/A9) |
| first_submitted_at | TIMESTAMPTZ | yes | | **added** (D13: lateness on first submission) |
| approved_at | TIMESTAMPTZ | yes | | |

Constraints: UNIQUE (project_id, sequence_no).

### 5.6 gateway_callbacks (NEW, A5) - A5 / V5
`callback_id BIGINT PK`, `gateway TEXT`, `gateway_txn_id VARCHAR(100)`, `raw_payload JSONB NOT NULL`, `signature_valid BOOLEAN NOT NULL`, `received_at TIMESTAMPTZ`, `processed_at TIMESTAMPTZ NULL`, `result VARCHAR(60)`. Index (gateway_txn_id). Raw payload must not contain card numbers (gateways do not send them; drop if present).

### 5.7 deliverables (PDF) - A6 / V6

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| deliverable_id | BIGINT identity | no | PK | |
| milestone_id | BIGINT | no | FK milestones | |
| version_no | INT | no | | 1, 2, 3... |
| file_url | VARCHAR(255) | yes | | Storage key |
| preview_url | VARCHAR(255) | yes | | Demo video / live link |
| is_source_released | BOOLEAN | no | default false | True after stage approved |
| scan_status | TEXT | no | CHECK IN (pending, clean, flagged) | Virus + contact scan |
| note | TEXT | yes | | Moderated |
| client_feedback | TEXT | yes | | On revision request |
| submitted_at | TIMESTAMPTZ | no | default now() | |
| auto_approve_at | TIMESTAMPTZ | no | | submitted_at + 7 days (D14) |

Constraints: UNIQUE (milestone_id, version_no). Index: (auto_approve_at).

### 5.8 payouts (PDF) - A6 / V6

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| payout_id | BIGINT identity | no | PK | |
| milestone_id | BIGINT | no | FK milestones, UNIQUE | One payout per stage |
| developer_id | BIGINT | no | FK users | |
| account_id | BIGINT | no | FK payout_accounts | |
| gross_amount | NUMERIC(12,2) | no | | Stage amount |
| penalty_deduction | NUMERIC(12,2) | no | default 0 | |
| commission | NUMERIC(12,2) | no | | 10% of (gross - penalty) |
| tax_withheld | NUMERIC(12,2) | no | default 0 | **added by A11** (D23) |
| net_amount | NUMERIC(12,2) | no | CHECK >= 0 | |
| status | TEXT | no | CHECK IN (scheduled, processing, awaiting_manual, paid, failed) | `awaiting_manual` **added** (D25) |
| scheduled_for | DATE | no | | Approval date + 1 business day (D6) |
| attempts | SMALLINT | no | default 0 | **added**: retries |
| paid_at | TIMESTAMPTZ | yes | | |
| gateway_txn_id | VARCHAR(100) | yes | UNIQUE WHERE NOT NULL | |

Index: (status, scheduled_for).

### 5.9 public_holidays (NEW, D6) - A6 / V6
`holiday_date DATE PK`, `name VARCHAR(100) NOT NULL`, `verified BOOLEAN NOT NULL DEFAULT false`, `updated_by FK users`. Admin-editable; Islamic holidays verified each year.

### 5.10 reliability_events (NEW, D15) - A6 / V6
`event_id BIGINT PK`, `developer_id FK users`, `delta SMALLINT NOT NULL`, `reason TEXT CHECK IN (on_time_stage, late_stage, developer_cancellation, abandonment, admin_adjustment)`, `source_type VARCHAR(30)`, `source_id BIGINT`, `created_at`. Append-only; `developer_profiles.reliability_score` = clamp(100 + sum(delta), 0, 100).

### 5.11 invoices (NEW, D24) - A11 / V10

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| invoice_id | BIGINT identity | no | PK | Immutable |
| number | VARCHAR(30) | no | UNIQUE | INV-YYYY-000001 / PAY-YYYY-000001 |
| series | TEXT | no | CHECK IN (INV, RFD, PAY) | client payment / refund / payout |
| type | TEXT | no | CHECK IN (client_payment, client_refund, developer_payout) | |
| user_id | BIGINT | no | FK users | Owner |
| project_id | BIGINT | no | FK projects | |
| source_payment_id | BIGINT | yes | FK payments, UNIQUE | Idempotency |
| source_payout_id | BIGINT | yes | FK payouts, UNIQUE | Idempotency |
| lines | JSONB | no | | Snapshot incl. tax lines |
| subtotal | NUMERIC(12,2) | no | | |
| tax_total | NUMERIC(12,2) | no | default 0 | |
| total | NUMERIC(12,2) | no | | |
| pdf_storage_key | VARCHAR(255) | yes | | Private |
| issued_at | TIMESTAMPTZ | no | default now() | |

### 5.12 invoice_sequences (NEW, D24) - A11 / V10
`series TEXT`, `year SMALLINT`, `last_number INT NOT NULL DEFAULT 0`, PK (series, year). Incremented with SELECT ... FOR UPDATE (gap-free).

### 5.13 manual_payout_records (NEW, D25) - A12 / V11
`record_id BIGINT PK`, `payout_id FK payouts`, `admin_id FK users`, `action TEXT CHECK IN (reveal_account, mark_paid, mark_failed)`, `transfer_reference VARCHAR(100)`, `transfer_date DATE`, `receipt_storage_key VARCHAR(255)`, `reason VARCHAR(255)`, `created_at`. Every account-number reveal is a row (audited).

---

## 6. Messaging (A7)

### 6.1 conversations (PDF) - A7 / V7
`conversation_id BIGINT PK`, `project_id FK projects`, `client_id FK users`, `developer_id FK users`, `created_at`. UNIQUE (project_id, developer_id).

### 6.2 messages (PDF) - A7 / V7

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| message_id | BIGINT identity | no | PK | |
| conversation_id | BIGINT | no | FK conversations | |
| sender_id | BIGINT | no | FK users | |
| body | TEXT | no | | Original text |
| delivered_body | TEXT | yes | | Admin may edit |
| status | TEXT | no | CHECK IN (auto_delivered, pending_review, approved, rejected) | |
| risk_score | NUMERIC(4,3) | yes | CHECK 0..1 | |
| route | TEXT | no | CHECK IN (direct, admin_queue) | |
| flag_categories | TEXT[] | yes | | **changed** from VARCHAR(150) |
| moderation_layers | TEXT | no | CHECK IN (layer1_only, layer1_and_ai) | **added** (D8 fallback visible) |
| created_at | TIMESTAMPTZ | no | default now() | |
| delivered_at | TIMESTAMPTZ | yes | | |

Indexes: (status, created_at), (conversation_id).

### 6.3 message_moderations (PDF) - A7 / V7
`moderation_id BIGINT PK`, `message_id FK messages`, `admin_id FK users`, `action TEXT CHECK IN (approve, edit_approve, reject)`, `reason VARCHAR(255)` (shown on reject), `strike_added BOOLEAN DEFAULT false` (**added**), `acted_at TIMESTAMPTZ`.

### 6.4 user_strikes (PDF) - A7 / V7
`strike_id BIGINT PK`, `user_id FK users`, `reason TEXT CHECK IN (contact_sharing, cancellation, abandonment)`, `source_type VARCHAR(30) CHECK IN (message, cancellation, project)`, `source_id BIGINT`, `created_at`, `expires_at TIMESTAMPTZ NOT NULL` (created + 6 months). Index (user_id, expires_at).

---

## 7. Rules & Penalties (A8, A11)

### 7.1 penalty_rules (PDF) - A8 / V8

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| rule_id | INT identity | no | PK | |
| rule_type | TEXT | no | CHECK IN (late_delivery, client_cancellation, developer_cancellation, abandonment) | |
| phase | TEXT | yes | CHECK IN (before_start, after_start) | **added**: client cancellation table has two columns (FR-CAN-01..03) |
| from_value | INT | no | | Late day or offence number |
| to_value | INT | yes | | NULL = no end |
| percent | NUMERIC(5,2) | no | | Per day (late) or flat |
| applies_to | TEXT | yes | CHECK IN (stage_amount, agreed_price, in_progress_stage) | **added**: base of the percent |
| max_percent | NUMERIC(5,2) | yes | | Cap, e.g. 25.00 |
| adds_strike | BOOLEAN | no | default false | **added** |
| triggers_review | BOOLEAN | no | default false | **added** |
| is_active | BOOLEAN | no | default true | |

Seed (late_delivery): (1-3, 2.00, stage_amount, max 25.00), (4-NULL, 3.00, stage_amount, max 25.00). Seed client_cancellation per FR-CAN-01..03 (exact rows depend on Q1).

### 7.2 penalties (PDF) - A8 / V8
`penalty_id BIGINT PK`, `user_id FK users` (who pays), `project_id FK projects`, `milestone_id FK milestones NULL`, `type TEXT CHECK IN (late_delivery, client_cancellation, developer_cancellation)`, `amount NUMERIC(12,2) CHECK >= 0`, `beneficiary_id FK users NULL` (NULL = platform), `status TEXT CHECK IN (applied, waived, reversed)`, `created_at`.

### 7.3 extension_requests (PDF) - A8 / V8
`request_id BIGINT PK`, `milestone_id FK milestones`, `requested_days SMALLINT CHECK 1..7`, `reason TEXT` (moderated), `status TEXT CHECK IN (pending, accepted, rejected)`, `decided_at TIMESTAMPTZ NULL` (**added**), `created_at` (must be before the stage due date - service).

### 7.4 cancellations (PDF) - A8 / V8

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| cancellation_id | BIGINT identity | no | PK | |
| project_id | BIGINT | no | FK projects | |
| cancelled_by | BIGINT | no | FK users | |
| by_role | TEXT | no | CHECK IN (client, developer) | |
| phase | TEXT | no | CHECK IN (before_start, after_start) | **added** |
| offence_no | SMALLINT | no | CHECK >= 1 | Snapshot of the derived count (D3) |
| stage_at_cancel | SMALLINT | yes | | |
| penalty_id | BIGINT | yes | FK penalties | NULL on first offence |
| reason | TEXT | yes | | |
| created_at | TIMESTAMPTZ | no | default now() | Offence counting: last 12 months |

Index: (cancelled_by, created_at).

### 7.5 tax_settings (NEW, D23) - A11 / V10
`tax_id INT PK`, `tax_type TEXT CHECK IN (sales_tax_on_fees, withholding_on_payouts)`, `rate NUMERIC(5,2) NOT NULL DEFAULT 0`, `is_active BOOLEAN NOT NULL DEFAULT false`, `effective_from TIMESTAMPTZ`, `notes VARCHAR(255)`, `updated_by FK users`.

---

## 8. Feedback, support, notifications (A9)

### 8.1 notifications (PDF) - A9 / V9

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| notification_id | BIGINT identity | no | PK | |
| user_id | BIGINT | no | FK users | Receiver |
| type | TEXT | no | CHECK IN (new_bid, bid_accepted, bid_rejected, offer, message, milestone, payment, payout, penalty, dispute, strike, account_status, kyc, system) | **extended** from PDF list (FR-NOT-01 events) |
| title | VARCHAR(150) | no | | |
| link | VARCHAR(255) | yes | | Screen to open |
| is_read | BOOLEAN | no | default false | |
| created_at | TIMESTAMPTZ | no | default now() | |

Index: (user_id, is_read).

### 8.2 notification_preferences (NEW, A9) - A9 / V9
`user_id FK users`, `notification_type TEXT`, `in_app BOOLEAN DEFAULT true`, `email BOOLEAN DEFAULT true`, PK (user_id, notification_type). Security emails cannot be disabled (service).

### 8.3 disputes (PDF) - A9 / V9

| Column | Type | Null | Constraint | Notes |
|---|---|---|---|---|
| dispute_id | BIGINT identity | no | PK | |
| project_id | BIGINT | no | FK projects | |
| milestone_id | BIGINT | yes | FK milestones | **added** (A9): frozen stage |
| raised_by | BIGINT | no | FK users | |
| reason | TEXT | no | | |
| status | TEXT | no | CHECK IN (open, under_review, resolved) | |
| resolution | TEXT | yes | CHECK IN (release_to_developer, refund_client, split) | |
| developer_percent | NUMERIC(5,2) | yes | CHECK 0..100 | **added**: for split |
| resolution_note | TEXT | yes | | **added** |
| resolved_by | BIGINT | yes | FK users | |
| created_at | TIMESTAMPTZ | no | default now() | **added** |
| resolved_at | TIMESTAMPTZ | yes | | |

### 8.4 reviews (PDF) - A9 / V9
`review_id BIGINT PK`, `project_id FK projects`, `reviewer_id FK users NULL` (NULL for system-generated; final choice in ADR 0006, A8), `reviewee_id FK users`, `rating SMALLINT CHECK 1..5`, `comment TEXT` (moderated), `is_system_generated BOOLEAN DEFAULT false`, `created_at`. UNIQUE (project_id, reviewer_id, reviewee_id) (D5 / C9).

### 8.5 support_tickets (PDF) - A9 / V9
`ticket_id BIGINT PK`, `user_id FK users NULL` (NULL = visitor contact form, F7), `contact_email VARCHAR(150) NULL` (visitors only, **added**), `subject VARCHAR(150)`, `body TEXT`, `status TEXT CHECK IN (open, answered, closed)`, `created_at`, `updated_at` (**added**).

### 8.6 support_ticket_replies (NEW, A9) - A9 / V9
`reply_id BIGINT PK`, `ticket_id FK support_tickets`, `author_id FK users`, `body TEXT NOT NULL`, `created_at`.

### 8.7 chatbot_logs (PDF) - A9 / V9
`log_id BIGINT PK`, `user_id FK users`, `question TEXT`, `answer TEXT`, `model_version VARCHAR(30)` (**added**), `created_at`. Contact details are stripped before storing.

---

## 9. System (A13)

### 9.1 alerts (NEW, A13) - A13 / V12
`alert_id BIGINT PK`, `type VARCHAR(60) NOT NULL`, `severity TEXT CHECK IN (info, warning, critical)`, `message VARCHAR(500)`, `entity_type VARCHAR(40)`, `entity_id BIGINT`, `dedup_key VARCHAR(150)`, `created_at`, `acknowledged_by FK users NULL`, `acknowledged_at TIMESTAMPTZ NULL`. Index (dedup_key, created_at), (acknowledged_at).

---

## 10. Relationships (from PDF section 4, plus new tables)

All PDF relationships are kept. Changes: `reviews.reviewer_id` may be NULL (system review). New relationships: users 1:N refresh_tokens, legal_acceptances, email_outbox, kyc_documents, reliability_events, notification_preferences, alerts (acknowledged_by); projects 1:N project_status_history, estimate_overrides, invoices; milestones 1:N disputes (optional); payouts 1:N manual_payout_records; payouts 1:0..1 invoices; payments 1:0..1 invoices; support_tickets 1:N support_ticket_replies.

## 11. Important constraints and indexes (PDF section 7, PostgreSQL form)

| Where | Rule | Why |
|---|---|---|
| bids | UNIQUE (project_id, developer_id) | One bid per developer per project |
| reviews | UNIQUE (project_id, reviewer_id, reviewee_id); CHECK rating 1..5 | D5 |
| conversations | UNIQUE (project_id, developer_id) | One thread per pair per project |
| projects | CHECK budget_min <= budget_max; band CHECK when open | Valid, realistic budget |
| projects | accepted_bid_id belongs to the same project | Enforced in BiddingService |
| wallets | UNIQUE (user_id); CHECK balances >= 0 | No negative money |
| milestones | SUM(payment_percent) per project = 100 (service); UNIQUE (project_id, sequence_no) | Stages equal the agreed price |
| deliverables | UNIQUE (milestone_id, version_no); INDEX (auto_approve_at) | History; auto-approve job |
| bid_offers | amount between allowed_min and allowed_max (service); INDEX (status, expires_at) | Band; expiry job |
| payouts | UNIQUE (milestone_id); INDEX (status, scheduled_for) | One payout per stage; payout job |
| project_assignments | UNIQUE (project_id) WHERE ended_at IS NULL | One active developer |
| user_strikes | INDEX (user_id, expires_at) | Active strike count |
| payout_accounts, kyc_documents | encrypted BYTEA | Never plain text |
| users | UNIQUE lower(email); UNIQUE phone WHERE NOT NULL | No duplicate accounts |
| developer_profiles | UNIQUE lower(username) | Unique public handle |
| projects | INDEX (status, category_id), INDEX (client_id), GIN (search_vector) | Browse / My Projects / search |
| bids | INDEX (project_id, status), INDEX (developer_id) | Bids view / My Bids |
| messages | INDEX (status, created_at), INDEX (conversation_id) | Moderation queue |
| notifications | INDEX (user_id, is_read) | Unread count |
| payments | INDEX (project_id); UNIQUE (gateway_txn_id) WHERE NOT NULL | History; idempotent callbacks |
| invoices | UNIQUE number; UNIQUE source_payment_id / source_payout_id | Gap-free, idempotent |
| All FKs | ON DELETE RESTRICT; users soft-deleted | Never lose payment or project history |

## 12. Entity classes and repositories

The PDF maps each table to one entity class (business layer) and one repository (data layer), e.g.
`projects -> Project (publish(), cancel(), acceptBid(bid), complete(), raiseDispute()) -> ProjectRepository`.
Keep that mapping and those method names where they fit; the full list is in
`docs/source/4_Database_Architecture.pdf` section 5. The Layered Architecture PDF groups them into 13
repositories (User, Project, Assignment, Bid, Message, Milestone, Payment, Payout, Rules, Penalty, Review,
Dispute, Notification); either granularity is fine as long as business code depends only on interfaces.

## 13. Differences from the PDF (summary)

1. MySQL `ENUM`/`DATETIME`/`VARBINARY`/`TINYINT` -> PostgreSQL `TEXT + CHECK`/`TIMESTAMPTZ`/`BYTEA`/`SMALLINT`.
2. Phone verification removed, email OTP (D20); account statuses merged (C7, needs approval).
3. Counter columns removed (C8); reliability score integer from events (C10).
4. Status values added: projects (D19, D4), milestones `disputed`, payouts `awaiting_manual` (D25), payments `tax_withheld` + gateway `bank_transfer`/`mock`.
5. 18 new tables (inventory above).
6. Open questions that may change columns: Q1 (penalty_rules seed), Q4 (interests), Q5 (deadline_days), Q6 (due dates) - see `docs/requirements.md` section 17.
