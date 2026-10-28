# Decisions (single source of truth)

Copied from the SHARED CONTEXT BLOCK in `docs/PROMPT_PACK.md`. Changing any decision needs both founders' approval in a PR that updates this file **and** the pack.

| ID | Decision |
|---|---|
| D1 | Money: NUMERIC(12,2) in DB, BigDecimal in Java (HALF_UP, scale 2). Never double/float. Every money movement is its own append-only payments row. Balances are derived or reconciled from rows. |
| D2 | Abandonment by lateness: client may end the assignment once a stage is more than 7 days late AFTER the 24h grace (i.e. from late-day 8). Developer quitting after first submission = abandonment. |
| D3 | Client cancels: before any stage submitted -> 1st: warning + full refund incl. fee; 2nd: 5% of agreed price to developer; 3rd+: 10% + strike + account review. After work started -> same offence rules PLUS developer gets 25% of the in-progress stage on the 1st offence (and the 5%/10% compensation on later ones, per the requirements table). Offence number is DERIVED from the cancellations table (last 12 months), not from a counter column. |
| D4 | Payment window: accepting a bid puts the project in status awaiting_payment (new status). Other bids are rejected ONLY when payment succeeds. If 48h pass without payment, acceptance lapses: project returns to open, the accepted bid returns to pending. |
| D5 | Review uniqueness: UNIQUE(project_id, reviewer_id, reviewee_id) (reopened projects can have two developers). |
| D6 | Business days: Mon-Fri excluding rows in table public_holidays (admin-editable). |
| D7 | Payments are MOCKED first (MockPaymentGateway implementing PaymentGatewayPort). Real JazzCash/Easypaisa come later after merchant approval. Gateway callbacks are idempotent on gateway_txn_id. |
| D8 | Moderation layer 1 (regex/patterns, Roman Urdu keywords) lives in Java. Layer 2 (AI intent score) is called on the Python ai-service. If the AI service is down or slow (>800 ms), fall back to layer-1 only and send anything with a layer-1 flag to the admin queue. |
| D9 | AI service is called over HTTP using contracts/openapi/ai-service.yaml with header X-Internal-Api-Key. AI estimate runs as a background job, never inside a user request. |
| D10 | Jev AI is NOT integrated yet. Only an evaluation + ADR is allowed (task B5). |
| D11 | Contact details (email, phone, real name) of one side must NEVER appear in any API response seen by the other side. Developers are shown by username only. |
| D12 | Time: all timestamps stored as TIMESTAMPTZ (UTC). All business-time calculations (deadlines, 48h windows, late days, business days, payout dates, invoice dates) use APP_TIMEZONE = Asia/Karachi. Use an injectable Clock everywhere so tests can control time. |
| D13 | Late days: late_hours = submitted_at - due_at. If late_hours <= 24 (grace), late_days = 0. Otherwise late_days = ceil((late_hours - 24) / 24): every started day after the grace counts as a full day. Penalty is CUMULATIVE: days 1-3 add 2% each, day 4 onward adds 3% each, total capped at 25% of the stage amount. So day 1 = 2%, day 2 = 4%, day 3 = 6%, day 4 = 9%, day 5 = 12% ... cap 25%. The worked example "two days late" = late_days 2 = 4% = PKR 1,600 on a PKR 40,000 stage. Lateness is measured on the FIRST submission of the stage; revisions requested by the client do not add lateness. |
| D14 | Auto-approve: 7 CALENDAR days (168h) after the latest submitted version of the stage, unless the stage is disputed or a revision was requested after that version. |
| D15 | Reliability score (developers): integer 0-100, starts at 100, stored on developer_profiles and changed only by reliability_events rows (append-only; score = clamp(100 + sum of deltas, 0, 100)). Default deltas live in the penalty_rules/settings data (not code): late stage -5, developer cancellation (2nd+) -10, abandonment -20, stage approved on time +2. Shown to clients as a number. |
| D16 | Account review: users.account_status values active \| under_review \| suspended \| closed. under_review: user cannot place bids, post new projects or start new conversations; work already in progress continues (messages, deliveries, payouts); admin must restore (active) or suspend. suspended: login allowed only to see data and support tickets; payouts already earned still paid. |
| D17 | Reopen pricing: when a reopened project's new agreed price for the remaining stages is accepted, difference = new_agreed_remaining - escrow_still_held. If difference > 0 the client pays it through the normal payment window (D4). If difference < 0 the extra amount is refunded to the client as a refund payments row on acceptance. Client fee applies only to the positive difference. |
| D18 | Wallets: payments rows are the ONLY source of truth. wallets is a cached summary updated in the same transaction; a reconciliation job compares it with the payments rows daily and raises an alert on any mismatch (it never silently "fixes" money). |
| D19 | Project statuses: draft, estimate_pending, estimate_failed, open, awaiting_payment, in_progress, under_review, completed, cancelled, disputed. estimate_failed = AI failed 3 times; admin sets the band manually, then the project can go to draft/open. |
| D20 | Verification: EMAIL OTP only (no SMS in Phase 1). Email sent through EmailPort: SmtpEmailAdapter configured for Gmail SMTP (smtp.gmail.com, port 587, STARTTLS, Gmail App Password; requires 2-Step Verification on that Gmail account) in prod; Mailpit (local SMTP catcher) in dev and test. Gmail has a daily sending limit (verify the current number in Google's official docs; do not hard-code it): keep EMAIL_DAILY_LIMIT as config, count sends per day, alert admin at 80%, and queue (not drop) non-OTP emails beyond the limit. EmailPort must be swappable later (Brevo, Amazon SES) without touching services. Phone number is collected as an optional field, NOT verified, and never shown to the other side (D11). Login is by email + password. |
| D21 | First admin: there is no admin sign-up. On startup, if no admin exists and ADMIN_BOOTSTRAP_EMAIL + ADMIN_BOOTSTRAP_PASSWORD are set, create one admin (must change password on first login), audit it, and log only "bootstrap admin created" (never the password). Further admins are created by an admin. |
| D22 | Scheduled jobs: every scheduled job (expiry, payment window, auto-approve, payouts, reconciliation, email queue) uses ShedLock with a DB lock table so it never runs twice, even if two backend instances run. Every job is idempotent. |
| D23 | Tax: tax rules are DATA in table tax_settings (e.g. sales tax on platform fees/commission, withholding tax on developer payouts), default rates 0% and is_active=false until a tax advisor confirms them. Never hard-code a tax rate. Every invoice shows tax lines (even when 0). |
| D24 | Invoices: every client payment, client-fee refund and developer payout gets an invoice/receipt with a sequential, gap-free number per series (INV-YYYY-000001, PAY-YYYY-000001), stored, immutable, downloadable as PDF by its owner and admin only. |
| D25 | Real payouts: until a disbursement API is approved, payouts run in MANUAL mode: due payouts appear in an admin "manual payouts" queue; admin transfers via bank/JazzCash outside the platform and marks each payout paid with the transfer reference number (audited). Switching to an API is a config change of PayoutPort (PAYOUT_MODE=manual\|api). |
| D26 | Developer KYC: developers must pass KYC (CNIC number + front/back image + selfie optional) before verification_status can become verified. CNIC number encrypted like payout accounts, images stored privately, visible only to admins, never returned to clients. Delete KYC images 90 days after the account closes unless a dispute or legal hold exists. |
| D27 | Legal pages: Terms of Service, Privacy Policy, Refund & Cancellation Policy, and Developer Agreement are versioned content (docs/legal/*.md, rendered by the frontend). Sign-up stores the accepted version and timestamp. Text is a DRAFT until reviewed by a lawyer (L1); the site shows nothing that claims legal approval. |

## Conflicts with the source PDFs

The source PDFs in `docs/source/` disagree with some decisions (tech stack, phone OTP, when other bids are
rejected, status lists, account statuses, counters, review uniqueness, deliverable contact scan). All
conflicts and which side wins are listed in `docs/requirements.md` section 16 (C1-C16). Open questions that
may change a decision are in section 17 (Q1-Q10).

## Change log

| Date | Decision | Change | Approved by |
|---|---|---|---|
| 2026-10-28 | D1-D27 | Initial version (pack v2) | Founder A, Founder B |
