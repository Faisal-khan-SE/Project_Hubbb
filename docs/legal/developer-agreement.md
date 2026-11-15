# Developer Agreement

> **DRAFT - not legally reviewed.** Full draft text is written in task F7 and replaced with lawyer-approved text in L1 (D27).

- Version: 0.0.0-draft
- Last updated: 2026-10-28

---

*Section headings added in task G1b. Full draft text is added in task F7.*

## 1. Eligibility

*(Covers: must be a natural person or sole proprietor based in Pakistan; must be 18 or older; must complete KYC (CNIC + images) and have it approved by an admin before bidding (D26, FR-KYC-01); must maintain a verified payout account (FR-PAY-04); must keep the Developer profile accurate and up to date.)*

## 2. KYC and Identity Verification

*(Covers: developer submits CNIC number and front/back images — optional selfie; CNIC number stored encrypted (AES-256-GCM, D26); images stored privately, visible to admins only; KYC re-submission allowed after rejection; a matching CNIC on two accounts is flagged for admin review; KYC images and encrypted number deleted 90 days after account closure unless a legal hold exists (D26).)*

## 3. Developer Responsibilities

*(Covers: accurate project proposals and estimates; delivering within the agreed timeline; submitting genuine, original deliverables for each stage; not sharing contact details through any platform channel — messages, notes, deliverable text, or filenames; complying with the moderation rules (D8, FR-MSG-05); not attempting to move business off-platform.)*

## 4. Project Delivery and Deadlines

*(Covers: three default stages — SRS 20%, Working Demo 40%, Source Code 40% (adjustable before Stage 1 starts); only one stage active at a time; each submission is a new version; lateness measurement — first submission against the due date (D13); 24-hour grace period; deadline extension requests — up to 7 days before the due date, requires client acceptance (FR-LATE-01); abandonment definition and consequences (D2, FR-ABN-01, FR-ABN-02).)*

## 5. Commission, Fees, and Payouts

*(Covers: 10% platform commission deducted from each stage payout after any late penalty; commission rate fixed at bid acceptance — later changes do not affect the running project (FR-PAY-03); late penalty deducted from stage payout and returned to client — not platform revenue (D13, BR-10); payouts scheduled for next business day after stage approval (BR-03, D6); payouts currently manual — admin processes and records transfer reference (D25, FR-PAY-09); payout account must remain valid and verified; if a payout fails, developer is notified and admin retries.)*

## 6. Moderation and Prohibited Content

*(Covers: all messages, proposal text, offer notes, extension reasons, review comments, and deliverable notes are checked by the moderation bot (D8, FR-MSG-05); prohibited: phone numbers, emails, social media handles, links, payment app details, or any other contact information; also prohibited: offensive, discriminatory, or legally sensitive content; penalties for violations — strikes (1 warning, 2 messaging locked 24h, 3 account under_review, D16, FR-MSG-06).)*

## 7. Strikes, Reliability Score, and Account Review

*(Covers: reliability score starts at 100; changes only via reliability_events (D15); deltas — late stage −5, cancellation 2nd+ −10, abandonment −20, on-time stage approval +2; strikes expire after 6 months; three abandonments in 12 months trigger account under_review (FR-ABN-02); account_status values and their restrictions — active / under_review / suspended / closed (D16, FR-ACC-01).)*

## 8. Intellectual Property and Deliverables

*(Covers: developer represents that they own or have the right to deliver all submitted work; on Stage 3 approval and full payment, ownership of the deliverable code transfers to the client per the agreed scope; the platform does not claim ownership; if deliverables include open-source components, the developer must disclose the licences; the client's project brief remains the client's IP.)*

## 9. Cancellation by Developer

*(Covers: cancellation within 24 hours of acceptance and before first submission — 1st offence warning, 2nd+ strike and −10 reliability (FR-CAN-04); after first submission, quitting counts as abandonment (FR-CAN-06, D2); project returns to open and client's escrow is held or refunded on request (D35, FR-CAN-05).)*

## 10. Representations and Warranties

*(Covers: developer has the skills and capacity to complete the project; deliverables are original or properly licensed; developer is not simultaneously offering the same project to a competing platform while under contract here; compliance with applicable Pakistani law.)*

## 11. Tax Obligations

*(Covers: developer is responsible for their own income tax obligations; the platform may withhold tax on payouts if required by law and confirmed by a tax advisor (D23, FR-TAX-01); withholding tax (if activated) is shown on the payout receipt; developer should consult a tax advisor regarding FBR obligations.)*

## 12. Governing Law (Pakistan)

*(Covers: this Agreement is governed by the laws of Pakistan; any dispute between the developer and the platform shall be subject to Pakistan jurisdiction; to be reviewed by a lawyer before launch — L1.)*

## 13. Changes to This Agreement

*(Covers: changes are version-controlled; developers must accept the new version before continuing to bid; accepted version and timestamp stored (D27); material changes will be communicated by email.)*
