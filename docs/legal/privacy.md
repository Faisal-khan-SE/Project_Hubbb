# Privacy Policy

> **DRAFT - not legally reviewed.** Full draft text is written in task F7 and replaced with lawyer-approved text in L1 (D27).

- Version: 0.0.0-draft
- Last updated: 2026-10-28

---

*Section headings added in task G1b. Full draft text is added in task F7.*

## 1. Data We Collect

*(Covers: account data — name, email, optional phone number; profile data — username, skills, bio; KYC data — CNIC number (encrypted), CNIC images, optional selfie (admin-visible only, D26); payout account — encrypted account number, type, masked last 4 digits; project data — title, description, attachments; message content; payment references; device and usage data for session management; legal acceptance records — document version and timestamp (D27).)*

## 2. How We Use Your Data

*(Covers: operating the marketplace — matching projects, verifying identities, processing payments; enforcing the moderation policy; calculating fees and penalties; sending platform notifications and OTP emails (D20); no use for advertising; no sale of personal data to third parties.)*

## 3. Data We Never Share Between Parties

*(Covers: clients never see a developer's real name, email, or phone number; developers never see a client's real name, email, or phone number; developers appear by username only (D11, FR-CORE-01); KYC data is never returned to any client-facing API (D26).)*

## 4. KYC Data and Encryption

*(Covers: CNIC number encrypted at rest using AES-256-GCM with a random IV per value (D26); CNIC images stored in a private storage bucket; visible only to platform admins via short-lived presigned URLs; every admin view is audited; a keyed hash of the CNIC number is used to detect duplicates without decryption; KYC images and the encrypted number are deleted 90 days after account closure unless a dispute or legal hold exists (D26).)*

## 5. Email Communications and OTP

*(Covers: email OTP for verification and password reset — stored hashed, single-use, 10-minute expiry (D20); notification emails sent for important platform events; unsubscribe available for non-security emails; OTP codes never logged in production; Gmail SMTP used in production (D20); email volume tracked against a daily limit and an alert raised at 80% usage.)*

## 6. Payout Account Data

*(Covers: account numbers encrypted (AES-256-GCM); only the last 4 digits shown in the UI (D11); full number visible only to admins on an explicit reveal action — audited; payout data not shared with clients.)*

## 7. Data Retention and Deletion

*(Covers: account data retained while the account is active; on account closure, personal contact data is removed except records needed for payments and law (invoices, payment rows, audit logs retain pseudonymous IDs — not contact data); KYC images deleted 90 days post-closure unless a legal hold exists (D26); users may request a data export of their own data (JSON download).)*

## 8. User Rights

*(Covers: right to access own data; right to export (JSON download); right to deletion (account closure with the retention exceptions above); right to correct profile information; to exercise rights, use the platform Settings or a support ticket.)*

## 9. Security

*(Covers: passwords hashed with Argon2 or bcrypt (NFR-SEC-01); CNIC and payout numbers encrypted at application level (NFR-SEC-04); HTTPS required (NFR-SEC-02); rate limiting on login, OTP, and sensitive actions (NFR-SEC-05); presigned short-lived file download URLs (NFR-SEC-06); input validation on all requests (NFR-SEC-07); no PII in logs.)*

## 10. Cookies and Tracking

*(Covers: session tokens (JWT, stored in HTTP-only cookies or secure local storage); no third-party advertising cookies; if any analytics are added in the future, they will be listed here and the Privacy Policy version incremented.)*

## 11. Changes to This Policy

*(Covers: version-controlled; sign-up stores the accepted version; users will be prompted to accept a new version when it changes materially.)*

## 12. Governing Law (Pakistan)

*(Covers: this policy is governed by the laws of Pakistan; relevant regulations — data protection provisions under Pakistani law, to be confirmed with a lawyer in L1.)*

## 13. Contact

*(Covers: how to reach the platform for privacy queries — via support ticket.)*
