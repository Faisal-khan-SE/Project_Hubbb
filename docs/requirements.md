# Requirements

Converted in task G1a from `docs/source/1_Requirements_Specification.pdf` (v1.0, Oct 2026), with supporting
detail from `2_Technology_Stack.pdf`, `3_Development_Roadmap.pdf`, `4_Database_Architecture.pdf` and
`5_Layered_Architecture.pdf`.

**How to read this file**
- FR-IDs that exist in the PDF are kept **exactly** (FR-PAY-01, FR-AI-01, FR-BID-01, FR-MSG-01, FR-DEL-01, FR-LATE-01, FR-ABN-01, FR-AUTH-01, FR-PRJ-01, FR-NOT-01, FR-REV-01, FR-DSP-01, FR-SUP-01, FR-BOT-01, FR-ADM-01).
- Requirements that the PDF states **without** an ID (business-rule table, cancellation rules, core principles, moderation layers) get a new ID marked **[G1a-ID]**. Only the number is new; the text comes from the PDF.
- Requirements added by the prompt pack decisions (D12-D27) are in section 14 and marked with their decision.
- Where the PDF and `docs/decisions.md` disagree, the **decision wins** (see section 16 "Conflicts"). Changed text is marked *(changed by Dxx)*.
- Priority: **M** = must for launch, **S** = should for launch, **L** = later (after beta). Priorities are proposed in G1a; the PDF does not state them.
- Source column: PDF section number, or decision ID.

## 1. Roles

| Role | Description | Source |
|---|---|---|
| Client | Uploads projects, compares bids, negotiates, pays into escrow, approves stages, rates the developer. | PDF 1.1 |
| Developer | Browses projects, bids and negotiates, delivers stages, receives payouts, rates the client. | PDF 1.1 |
| Admin | Reviews flagged messages, verifies users, resolves disputes, manages fees and penalty rules. | PDF 1.1 |
| System (bots and jobs) | AI estimator, moderation bot, auto-approve job, payout job, penalty calculator. | PDF 1.1 |

## 2. Core principles

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-CORE-01 [G1a-ID] | **No direct contact.** Clients and developers never see each other's phone, email or real name. Developers appear by username. | System | M | PDF 1.2, D11 |
| FR-CORE-02 [G1a-ID] | **No direct payment.** The client pays the platform; the platform keeps its fees and pays the developer. | System | M | PDF 1.2 |
| FR-CORE-03 [G1a-ID] | **Fair price.** The AI estimate sets a band that the budget, bids and counter-offers must stay inside. | System | M | PDF 1.2 |
| FR-CORE-04 [G1a-ID] | **Pay as delivered.** Work is split into stages; each approved stage is paid out within one business day. | System | M | PDF 1.2 |
| FR-CORE-05 [G1a-ID] | **Rules as data.** Every fee and penalty number is stored in the database so admin can change it without new code. | Admin | M | PDF 1.2 |

## 3. Business rules (default values)

All values are defaults and editable by admin (tables `fee_settings`, `penalty_rules`, and `platform_settings` for non-money numbers - see database-design.md). The PDF notes these are proposed defaults to be reviewed after the beta.

| ID | Rule | Default value | Source |
|---|---|---|---|
| BR-01 [G1a-ID] | Client service fee | 5% of agreed price, added on top, minimum PKR 300. Covers gateway charges. | PDF 2 |
| BR-02 [G1a-ID] | Developer commission | 10% of each stage payout, calculated after any late penalty. | PDF 2 |
| BR-03 [G1a-ID] | Payout time | Next business day after a stage is approved (business days per D6, time zone per D12). | PDF 2, D6, D12 |
| BR-04 [G1a-ID] | Allowed budget band | Lowest: 80% of AI low estimate. Highest: 130% of AI high estimate. | PDF 2 |
| BR-05 [G1a-ID] | Allowed deadline | At least 70% of the AI's minimum days. | PDF 2 |
| BR-06 [G1a-ID] | Delivery stages | SRS 20%, Working demo 40%, Source code and handover 40%. | PDF 2 |
| BR-07 [G1a-ID] | Negotiation | Structured offers only (price + days), max 3 rounds per bid, each offer valid 48 hours. | PDF 2 |
| BR-08 [G1a-ID] | Auto-approve | A submitted stage is approved automatically if the client does not respond in 7 days (calendar days, D14). | PDF 2, D14 |
| BR-09 [G1a-ID] | Late grace period | 24 hours after the stage due date. | PDF 2 |
| BR-10 [G1a-ID] | Late penalty | 2% of stage amount per day for days 1-3, 3% per day from day 4, capped at 25%. Paid to the client as a discount. Counting per D13. | PDF 2, 8.1, D13 |
| BR-11 [G1a-ID] | Abandonment | Developer quits, or is 7 days late (after grace) and the client chooses to end the assignment (from late-day 8). | PDF 2, FR-LATE-02, D2 |
| BR-12 [G1a-ID] | Client cancellation | 1st: warning only. 2nd: 5% of agreed price to developer. 3rd+: 10% plus strike and account review. Full table in 9.3. | PDF 2, 8.3, D3 |
| BR-13 [G1a-ID] | Developer cancellation before start | 1st: warning. 2nd+: strike and reliability score drop (10 points). | PDF 2, 8.4 |
| BR-14 [G1a-ID] | Strikes | 1: warning. 2: messaging locked 24 hours. 3: account review (D16). Strikes expire after 6 months. | PDF 2, D16 |
| BR-15 [G1a-ID] | Offence counting | Cancellations are counted over the last 12 months, derived from the `cancellations` table (D3). | PDF 2, D3 |

## 4. Money flow and payments

Money never goes from the client to the developer directly. The client pays the full agreed price plus the
client fee into platform escrow when accepting a bid. Each stage's share stays held until the client approves
that stage; it is then paid to the developer's verified payout account on the next business day, after
deducting any late penalty and the 10% commission. (PDF 3)

### 4.1 Worked example (mandatory test fixture, task A8)

Agreed price PKR 100,000. Stage 2 is delivered 2 days late (after the grace period). Stages 1 and 3 are on time.

| Step | Calculation | Amount (PKR) |
|---|---|---|
| Client pays at acceptance | 100,000 + 5% fee | 105,000 |
| Stage 1, SRS (20%) | 20,000 - 10% commission (2,000) | Developer gets 18,000 |
| Stage 2, Demo (40%) | 40,000 - late penalty 2 days x 2% = 1,600 | Client refunded 1,600 |
| | 38,400 - 10% commission (3,840) | Developer gets 34,560 |
| Stage 3, Source code (40%) | 40,000 - 10% commission (4,000) | Developer gets 36,000 |
| **Totals** | Developer 88,560 · Platform 14,840 (5,000 + 2,000 + 3,840 + 4,000) · Client net cost 103,400 (105,000 - 1,600) | 88,560 + 14,840 = 103,400 |

### 4.2 Payment requirements

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-PAY-01 | The client must pay the agreed price plus client fee within 48 hours of accepting a bid; otherwise the acceptance lapses and the bid returns to pending (project returns to open, D4). | Client, System | M | PDF 3.2, D4 |
| FR-PAY-02 | Supported methods: JazzCash, Easypaisa, debit/credit card through a gateway, and bank transfer (manually confirmed by admin). | Client, Admin | M | PDF 3.2 |
| FR-PAY-03 | Fees are fixed at acceptance time using the active `fee_settings` row; later fee changes do not affect running projects. | System | M | PDF 3.2 |
| FR-PAY-04 | A developer must add and verify a payout account before placing any bid. | Developer | M | PDF 3.2 |
| FR-PAY-05 | On stage approval the system schedules a payout for the next business day. Failed payouts are retried and both developer and admin are notified. (Manual payout mode until a disbursement API is approved, D25.) | System | M | PDF 3.2, D25 |
| FR-PAY-06 | Every money movement (deposit, fee, release, commission, refund, penalty, compensation) is recorded as its own `payments` row. | System | M | PDF 3.2, D1 |
| FR-PAY-07 | The client fee is refundable only if the project is cancelled before Stage 1 is approved. | System | M | PDF 3.2 |
| FR-PAY-08 | Gateway callbacks are idempotent: the same transaction reference can never be applied twice. | System | M | PDF 3.2, D7 |

## 5. AI price and time estimation

When a client uploads a project, the AI reads the title, description, category, skills and attached files and
returns a market price range, a time range, a complexity level and the list of features it found. (PDF 4)

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-AI-01 | Version 1 uses a language model to extract features and complexity, then a formula based on admin-managed rates per category and feature. Version 2 (after about 300 completed projects) uses a model trained on real completed prices. | System, Admin | M (v1) / L (v2) | PDF 4 |
| FR-AI-02 | The client's `budget_min` must be at least 80% of the AI low price, and `budget_max` at most 130% of the AI high price. Values outside the band block publishing. | Client, System | M | PDF 4 |
| FR-AI-03 | The deadline must be at least 70% of the AI minimum days. | Client, System | M | PDF 4 |
| FR-AI-04 | If the client believes the estimate is wrong, they can send the project for admin review with a reason. Admin can override the band; the override is logged. | Client, Admin | M | PDF 4 |
| FR-AI-05 | The estimate is shown to developers on the project page, and recalculated for remaining stages when a project reopens. | Developer, System | M | PDF 4 |
| FR-AI-06 | Every estimate is stored with its model version so accuracy can be measured later. | System | M | PDF 4 |

## 6. Bidding and negotiation

Negotiation is structured: offers carry only a price and a number of days, plus an optional short note that
passes through the moderation bot. (PDF 5)

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-BID-01 | A developer can place one bid per project: amount, days, and a proposal (moderated). Bids must stay inside the allowed band. | Developer | M | PDF 5 |
| FR-BID-02 | The client can counter any bid. The developer can accept, counter or reject. Maximum 3 rounds per bid; each offer expires after 48 hours. | Client, Developer | M | PDF 5 |
| FR-BID-03 | Every offer must be inside the project's allowed band. The note is limited to 200 characters. | System | M | PDF 5 |
| FR-BID-04 | The client can negotiate with several developers at once, but can accept only one bid. | Client | M | PDF 5 |
| FR-BID-05 | Accepting a bid creates the stages from the delivery template and starts the payment window (FR-PAY-01). *(changed by D4: other bids are rejected and stages are created only when payment succeeds; until then the project is `awaiting_payment`.)* | Client, System | M | PDF 5, D4 |
| FR-BID-06 | The client sees bids side by side: amount, days, rating, reliability score, completed projects. No contact details. | Client | M | PDF 5 |

## 7. Messaging and the moderation bot

Every message is checked by a moderation bot first. Clean messages are delivered instantly; only suspicious
messages go to admin. (PDF 6)

### 7.1 How the bot decides

| ID | Layer / routing | What it does | Pri | Source |
|---|---|---|---|---|
| FR-MOD-01 [G1a-ID] | Layer 1: pattern rules | Pakistani phone formats (03XX, +92, 0092) including digits split by spaces, dots or dashes; numbers written as words in English or Roman Urdu; emails including "at the rate" and "dot com"; links; CNIC and IBAN patterns; keywords such as WhatsApp, Insta, Telegram, "number do", "direct payment", "bahar deal". | M | PDF 6.1, D8 |
| FR-MOD-02 [G1a-ID] | Layer 2: AI classifier | A language model scores the intent to move the conversation or payment off the platform, including disguised attempts. Fallback to layer 1 if the AI is down or slower than 800 ms (D8). | M | PDF 6.1, D8 |
| FR-MOD-03 [G1a-ID] | Layer 3: attachments | Text inside images and documents is extracted (OCR) and checked with layers 1 and 2. **Not yet covered by any task - see Open question Q3.** | S | PDF 6.1 |
| FR-MOD-04 [G1a-ID] | Routing | Risk below 0.3: delivered instantly. 0.3 to 0.7: held in admin queue, sender sees "under review". Above 0.7: held in admin queue; if admin confirms contact sharing, the sender gets a strike. | M | PDF 6.1 |

### 7.2 Messaging requirements

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-MSG-01 | One conversation per client-developer pair per project. Messaging is allowed from the first bid onward. | Client, Developer | M | PDF 6.2 |
| FR-MSG-02 | The admin queue must be reviewed within 2 hours between 9 am and 11 pm (Asia/Karachi). | Admin | S | PDF 6.2, D12 |
| FR-MSG-03 | Admin can approve, edit then approve, or reject with a reason that the sender sees. | Admin | M | PDF 6.2 |
| FR-MSG-04 | Both the original and delivered text are stored, with every admin decision. | System | M | PDF 6.2 |
| FR-MSG-05 | The same bot checks bid proposals, offer notes, extension reasons, review comments and deliverable files (deliverable scope: see Conflict C12). | System | M | PDF 6.2 |
| FR-MSG-06 | Strike policy: 1st warning, 2nd messaging locked for 24 hours, 3rd account sent for review (D16). Strikes expire after 6 months. | System, Admin | M | PDF 6.2, D16 |

## 8. Staged delivery

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-DEL-01 | Default stages: SRS (20%), Working demo with video and live link (40%), Source code and handover (40%). Client and developer can change percentages before work starts; they must total 100%. | Client, Developer | M | PDF 7 |
| FR-DEL-02 | Only one stage is active at a time; the next stage unlocks when the previous one is approved. | System | M | PDF 7 |
| FR-DEL-03 | Each submission is scanned for viruses and contact details before the client sees it. | System | M | PDF 7 |
| FR-DEL-04 | The client approves or requests a revision with written feedback. Each resubmission is saved as a new version. | Client, Developer | M | PDF 7 |
| FR-DEL-05 | A stage is auto-approved if the client does not respond within 7 days of submission (D14). | System | M | PDF 7, D14 |
| FR-DEL-06 | Source code becomes downloadable for the client only after the source-code stage is approved. | System | M | PDF 7 |

What each stage contains (PDF 4_Database_Architecture, section 3):

| Stage | Developer submits | Client sees | Payment |
|---|---|---|---|
| 1. SRS | SRS document (requirements, use cases, screens) | The full document | 20% |
| 2. Working demo | Demo video plus a live demo link | Video and live demo only, no source code | 40% |
| 3. Source code and handover | Source code zip, setup guide, final documents | Source zip, unlocked after approval | 40% |

## 9. Deadlines, penalties and cancellations

### 9.1 Late delivery

Each stage has its own due date. After a 24-hour grace period, the stage amount is reduced per late day. The
deduction goes back to the client as a discount; the platform keeps none of it. (PDF 8.1)

| Days late (after grace) | Penalty per day | Total at end of range |
|---|---|---|
| 1 to 3 | 2% of stage amount | 6% |
| 4 to 7 | 3% of stage amount | 18% |
| 8 and more | 3% of stage amount, up to the cap | 25% maximum |

The "total at end of range" column confirms the penalty is **cumulative**, exactly as D13 defines.

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-LATE-01 | A developer can request an extension (up to 7 days per stage) before the due date. If the client accepts, the due date moves and no penalty applies. | Developer, Client | M | PDF 8.1 |
| FR-LATE-02 | From day 8 late, the client may end the assignment, which counts as abandonment by the developer. | Client | M | PDF 8.1, D2 |
| FR-LATE-03 | Every late delivery lowers the developer's reliability score (D15). | System | M | PDF 8.1, D15 |

### 9.2 Developer abandons the project

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-ABN-01 | Approved stages stay paid. A submitted but unapproved stage goes to the client for a decision within 3 days (approve and pay, or reject and refund). An unsubmitted stage is not paid. | Client, System | M | PDF 8.2 |
| FR-ABN-02 | The system adds an automatic 1-star review marked "Project abandoned", a strike, and a 20-point reliability score drop. Three abandonments in 12 months suspend the account for review (D16 `under_review`). | System | M | PDF 8.2, D15, D16 |
| FR-ABN-03 | The project reopens automatically from the next stage. Completed stages and their deliverables stay attached and are visible to new bidders and the next developer. | System | M | PDF 8.2 |
| FR-ABN-04 | The AI recalculates price and time for the remaining stages. Remaining escrow stays held; if the new agreed price is higher, the client pays the difference (lower: refund, D17). | System, Client | M | PDF 8.2, D17 |

### 9.3 Client cancels after accepting a bid

Offence number = count of the client's cancellations in the last 12 months (derived, D3).

| ID | Offence | Before any stage submitted | After work has started |
|---|---|---|---|
| FR-CAN-01 [G1a-ID] | 1st | Warning only. Full refund including client fee. | Warning. Approved stages stay paid; a submitted stage must be approved or disputed; the developer gets 25% of the in-progress stage as compensation. |
| FR-CAN-02 [G1a-ID] | 2nd | 5% of agreed price paid to the developer as compensation. | Same as above, plus the 5% compensation. (See Open question Q1.) |
| FR-CAN-03 [G1a-ID] | 3rd and more | 10% of agreed price to the developer, a strike, and account review. | Same as above, plus the 10% compensation and strike. |

Source: PDF 8.3, D3. Actor: Client, System. Priority: M.

### 9.4 Developer cancels before starting

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-CAN-04 [G1a-ID] | Within 24 hours of acceptance and before submitting anything: the first time is a warning; from the second time, a strike and a 10-point reliability drop. | Developer, System | M | PDF 8.4 |
| FR-CAN-05 [G1a-ID] | After a developer cancels before starting, the project returns to Open for Bids and the client's escrow is held for the next developer or refunded on request. | Client, System | M | PDF 8.4 |
| FR-CAN-06 [G1a-ID] | After the first submission, quitting counts as abandonment (9.2). | System | M | PDF 8.4, D2 |

## 10. Other functional requirements

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-AUTH-01 | Sign up with name, email, optional phone and password; **email verified by OTP** *(changed by D20; PDF: phone verified by OTP)*. Role chosen at sign-up. Accepted legal document versions stored (D27). | Client, Developer | M | PDF 9, D20, D27 |
| FR-AUTH-02 | Login with **email** and password; password reset by **email OTP**; sessions expire and can be revoked. *(changed by D20; PDF: login with email or phone, OTP by SMS)* | All | M | PDF 9, D20 |
| FR-AUTH-03 | Developers must complete a profile (skills, experience, portfolio) and be verified by admin before bidding (verification requires approved KYC, D26). | Developer, Admin | M | PDF 9, D26 |
| FR-AUTH-04 | A one-time pop-up after client sign-up collects city and interests (Places). | Client | S | PDF 9, PDF 11 |
| FR-PRJ-01 | Client uploads a project: title, description, category, skills, files, budget range, deadline; can save as draft. | Client | M | PDF 9 |
| FR-PRJ-02 | My Projects shows every project with its status and the next action needed. | Client | M | PDF 9 |
| FR-NOT-01 | Real-time in-app notifications for bids, offers, messages, submissions, payouts, penalties; **email** for important ones *(changed by D20; PDF: email/SMS)*. | All | M | PDF 9, D20 |
| FR-REV-01 | After completion both sides can leave a 1-5 rating once; comments are moderated. | Client, Developer | M | PDF 9, D5 |
| FR-DSP-01 | Either side can raise a dispute; the current stage is frozen until admin decides (release, refund or split). Already-approved stages are not affected. | Client, Developer, Admin | M | PDF 9 |
| FR-SUP-01 | Contact screen opens support tickets to admin. | All | M | PDF 9 |
| FR-BOT-01 | Dashboard chatbot answers platform questions and helps clients write clearer project descriptions. | Client, Developer | S | PDF 9 |
| FR-ADM-01 | Admin panel: message queue, user verification, project review, payouts, disputes, fee and penalty settings, reports. | Admin | M | PDF 9 |
| FR-ADM-02 | Every admin and money action is written to the audit log. | System | M | PDF 9 |

## 11. Non-functional requirements

| ID | Area | Requirement | Source |
|---|---|---|---|
| NFR-SEC-01 | Security | Passwords hashed with Argon2 or bcrypt. | PDF 10 |
| NFR-SEC-02 | Security | HTTPS only. (Hosting is outside the current pack; enforced at deployment.) | PDF 10 |
| NFR-SEC-03 | Security | Role checks on every API call, plus ownership checks so a user can only load their own projects, bids and files. | PDF 10, Tech Stack 7 |
| NFR-SEC-04 | Security | Payout account numbers encrypted (AES-256, application level); also KYC numbers (D26). | PDF 10, DB 7, D26 |
| NFR-SEC-05 | Security | Rate limiting on login, OTP, messaging and bid placement. | PDF 10, Tech Stack 7 |
| NFR-SEC-06 | Security | Presigned, short-lived URLs for file downloads; source-code files locked until the stage is approved. | Tech Stack 7 |
| NFR-SEC-07 | Security | Input validation on every request; secrets only in environment variables, never in the repository; dependency scanning. | Tech Stack 7 |
| NFR-PRV-01 | Privacy | Contact details never returned by any API to the other side. | PDF 10, D11 |
| NFR-PRV-02 | Privacy | Personal data deletable on account closure, except records needed for payments and law. | PDF 10 |
| NFR-PRF-01 | Performance | Pages load in under 2 seconds on 4G. | PDF 10 |
| NFR-PRF-02 | Performance | Clean messages delivered in under 1 second. | PDF 10 |
| NFR-PRF-03 | Performance | AI estimate in under 20 seconds. | PDF 10 |
| NFR-AVL-01 | Availability | 99.5% monthly uptime target. | PDF 10 |
| NFR-AVL-02 | Availability | Daily database backups kept for 30 days. | PDF 10 |
| NFR-SCL-01 | Scalability | Stateless backend that can run several copies (jobs locked per D22). | PDF 10, D22 |
| NFR-SCL-02 | Scalability | Background jobs for AI, scanning, payouts and notifications; AI calls never inside a user request (D9). | PDF 10, D9 |
| NFR-MNY-01 | Money accuracy | All amounts stored as exact decimals; every movement traceable in the `payments` table. | PDF 10, D1 |
| NFR-USE-01 | Usability | Mobile-first web design; English interface with Urdu support planned (mobile apps after the web version is stable). | PDF 10, PDF 11 |
| NFR-MNT-01 | Maintainability | Layered architecture; fees and penalty numbers stored as data; automated tests for every business rule. | PDF 10 |
| NFR-MNT-02 | Maintainability | Business-layer test coverage stays above 80%; every endpoint documented in the OpenAPI file. | Roadmap 4 (Definition of Done) |

## 12. State machines (allowed transitions)

From `4_Database_Architecture.pdf` section 6. The Business layer must reject any other change. Where the
decisions add states, they are marked.

| Entity | Allowed transitions | Source |
|---|---|---|
| projects | draft -> estimate_pending -> draft (estimate ready) / estimate_failed *(D19)*; estimate_failed -> draft (admin sets band) *(D19)*; draft -> open (publish); open -> awaiting_payment (bid accepted) *(D4)*; awaiting_payment -> open (48 h lapse) / in_progress (paid) *(D4)*; in_progress -> under_review -> completed; open -> cancelled; in_progress -> disputed -> in_progress / completed / cancelled; in_progress -> open (reopen after abandonment) | DB 6, D4, D19, Q2 |
| bids | pending -> negotiating -> shortlisted -> accepted; pending / shortlisted -> rejected; pending / shortlisted -> withdrawn (by developer); accepted -> pending (payment window lapsed) *(D4)* | DB 6, D4 |
| bid_offers | pending -> accepted / countered / rejected / expired (48 hours); max 3 rounds per bid | DB 6 |
| messages | auto_delivered (risk < 0.3); pending_review -> approved / rejected (with reason) | DB 6 |
| milestones | locked -> in_progress (previous stage approved) -> submitted -> approved; submitted -> revision_requested -> submitted; submitted -> approved automatically after 7 days; in_progress / submitted -> disputed -> (admin decision) *(A6/A9)* | DB 6 |
| deliverables | scan_status: pending -> clean / flagged; is_source_released: false -> true when its stage is approved | DB 6 |
| payments | pending -> held (deposit succeeded) -> released; held -> refunded; pending -> failed | DB 6 |
| payouts | scheduled -> processing -> paid; processing -> failed -> scheduled (retry); scheduled -> awaiting_manual -> paid / failed *(D25)* | DB 6, D25 |
| project_assignments | active -> ended (completed / abandoned / cancelled_by_client / cancelled_by_developer) | DB 6 |
| extension_requests | pending -> accepted (due date moves) / rejected | DB 6 |
| disputes | open -> under_review -> resolved | DB 6 |

## 13. Key flows (summary)

From `5_Layered_Architecture.pdf` section 5. Detailed screens are in `docs/screens.md` (task G1b).

1. **Client uploads a project:** form -> ProjectService validates -> EstimationService runs the AI estimate (background job) and sets the band -> saved as draft -> client adjusts and publishes.
2. **Developer places a bid:** BiddingService checks project open, amount inside band, no earlier bid -> bid saved as pending -> notification.
3. **Negotiation:** NegotiationService checks band, 3-round limit, sets 48 h expiry -> accepted offer becomes the bid's final amount.
4. **Accept and pay:** project -> awaiting_payment; on payment success other bids rejected, stages created, assignment starts (D4).
5. **Message moderation:** layer 1 + AI score -> instant delivery or admin queue -> strike on confirmed contact sharing.
6. **Stage delivery and payout:** upload -> scan -> approve / revision / auto-approve -> penalty, commission, payout next business day -> next stage unlocks.
7. **Abandonment:** assignment ends as abandoned -> approved stages stay paid -> system review + strike -> reopen from next stage with AI re-estimate.

## 14. Requirements added by the prompt pack decisions

These are not in the PDFs. They come from `docs/decisions.md` and are needed so the traceability matrix (A10) covers them.

| ID | Requirement | Actor | Pri | Source |
|---|---|---|---|---|
| FR-TIME-01 | All business-time calculations use Asia/Karachi; timestamps stored in UTC. | System | M | D12 |
| FR-REL-01 | Reliability score 0-100, starts at 100, changed only through append-only events; deltas are data. | System | M | D15 |
| FR-ACC-01 | Account statuses active / under_review / suspended / closed with the restrictions defined in D16. | Admin, System | M | D16 |
| FR-ACC-02 | First admin created by a one-time bootstrap from environment variables; no admin sign-up. | System | M | D21 |
| FR-EML-01 | Email delivery through a swappable email port (Gmail SMTP in prod, Mailpit in dev/test), with an outbox, retries, daily-limit tracking and an alert at 80%. | System | M | D20 |
| FR-JOB-01 | Every scheduled job is locked (runs once even with several backend copies) and idempotent. | System | M | D22 |
| FR-TAX-01 | Tax rules (sales tax on fees, withholding on payouts) stored as data, 0% and inactive until confirmed by a tax advisor. | Admin | M | D23 |
| FR-INV-01 | Every client payment, client-fee refund and developer payout gets an immutable invoice/receipt with a gap-free sequential number, downloadable as PDF by its owner and admin only. | All | M | D24, PDF screen "Payment: invoices" |
| FR-PAY-09 | Manual payout mode: due payouts go to an admin queue; admin pays outside the platform and records the transfer reference. | Admin | M | D25 |
| FR-KYC-01 | Developers pass KYC (CNIC + images) before verification; data encrypted, admin-only, deleted 90 days after closure unless on hold. | Developer, Admin | M | D26 |
| FR-LEG-01 | Terms, Privacy, Refund & Cancellation Policy and Developer Agreement are versioned; acceptance stored at sign-up. | All | M | D27 |
| FR-OBS-01 | Alerts (email to admin) for money, delivery, email-limit, AI-down and scanning failures; error tracking with PII scrubbing. | System, Admin | M | Pack A13, Tech Stack 1 |
| FR-PUB-01 | Public pages (landing, how it works, pricing, FAQ, help, legal) and SEO; pricing numbers read from the backend. | Visitor | M | Pack F7 |

## 15. Assumptions and open items (from the PDF)

- **Legal check needed before launch.** Holding customer money in escrow may need a licensed payment partner or specific merchant terms under State Bank of Pakistan rules. Confirm with JazzCash/Easypaisa or a payment aggregator, and ideally a lawyer, before building the payment module. Merchant approval can take several weeks, so start early. (Tracked in L1.)
- All fee and penalty values are proposed defaults to be reviewed after the beta.
- The Places pop-up is assumed to collect city and interests (confirm - Q4).
- Mobile apps come after the web version is stable.

## 16. Conflicts (PDF vs decisions)

Rule: **the decisions in `docs/decisions.md` win** unless both founders change them.

| # | PDF says | Decision / pack says | Winner |
|---|---|---|---|
| C1 | Backend NestJS (TypeScript) + Prisma + Jest (Tech Stack 1-2). The PDF also lists Spring Boot as an equally good alternative. | Java 21 + Spring Boot 3 + Flyway + JUnit (ADR 0001). | Decision (ADR 0001). |
| C2 | Redis + BullMQ for cache, rate limits and job queues; Socket.IO for real time. | No Redis yet; DB-backed jobs with ShedLock (D22); in-memory rate limiting behind an interface; WebSocket (STOMP) or SSE. | Decision. Redis can be added later behind the same interfaces. |
| C3 | AI in phase 1 is an LLM API called through an adapter in the backend; Python FastAPI only in phase 2. | Separate Python FastAPI ai-service from day 1, called over HTTP (ADR 0002, D9). | Decision. |
| C4 | Phone verified by SMS OTP; login with email or phone; `users.is_phone_verified`; Email/SMS adapter. | Email OTP only (Gmail SMTP); login by email; phone optional and unverified; no SMS (D20). | Decision. |
| C5 | Accepting a bid immediately rejects all other bids and creates stages (FR-BID-05; Layered 5.4). | Other bids rejected and stages created only when payment succeeds; new status `awaiting_payment` (D4). | Decision. |
| C6 | `projects.status`: draft, open, in_progress, under_review, completed, cancelled, disputed. (Screens already mention "Awaiting Payment".) | Adds estimate_pending, estimate_failed, awaiting_payment (D19, D4). | Decision. |
| C7 | `account_status`: pending, active, blocked. | active, under_review, suspended, closed (D16). | Merge proposed in database-design.md: `pending_verification, active, under_review, suspended, closed` (blocked = suspended). Needs both founders' OK. |
| C8 | Counter columns `client_profiles.cancellation_count`, `developer_profiles.abandoned_count`. | Offence and abandonment counts derived from `cancellations` / `project_assignments` (D3). | Decision: columns removed. |
| C9 | `reviews` UNIQUE (project_id, reviewer_id). | UNIQUE (project_id, reviewer_id, reviewee_id) because reopened projects can have two developers (D5). | Decision. |
| C10 | `reliability_score DECIMAL(5,2)`. | Integer 0-100 derived from `reliability_events` (D15). | Decision. |
| C11 | `payments.gateway` enum jazzcash, easypaisa, card. FR-PAY-02 also lists bank transfer. | Adds `bank_transfer` (admin-confirmed) (pack A5). | Decision (fixes an internal PDF gap). |
| C12 | Every deliverable is scanned for phone numbers, emails and links (FR-DEL-03, FR-MSG-05). | Contact scan covers document text and notes only, not source-code files, which legitimately contain emails and URLs (pack A6). | Decision, but see Q3. |
| C13 | Roadmap: staging server, auto-deploy, Nginx, production setup in S0/S9. | Hosting, CD, SSL and domain are out of scope of the current pack. | Decision (deferred, not dropped). |
| C14 | Notifications by email/SMS (FR-NOT-01; Layered: NotificationService email and SMS). | Email only (D20). | Decision. |
| C15 | Layered: 4 external adapters (payment gateway, AI model, file storage, Email/SMS). | More ports: PaymentGatewayPort, PayoutPort, AiModelPort, FileStoragePort, EmailPort, VirusScanPort, AlertPort. | Decision (superset). |
| C16 | Roadmap: 20 weeks, 3-4 developers, 2-week sprints S0-S9. | 13 weeks, 2 founders, tasks G0-L1. | Decision (planning only; same module order). |

## 17. Decided questions

All ten open questions from G1a and conflict C7 were decided on 2026-10-28 and recorded as decisions
D28-D38 in `docs/decisions.md`. The answers follow common marketplace practice: the developer is never
penalised for the client's review time, the client is never charged beyond the escrow they paid, and nothing
about money happens automatically without the user's choice.

| # | Question | Decision | Affects |
|---|---|---|---|
| Q1 | Client cancels **after work started**, 2nd or 3rd offence: does the developer get 25% of the in-progress stage **plus** 5% / 10%? | **Yes, both** (as the PDF table reads). Compensation comes only from escrow still held and is capped at that amount; the client is never charged extra; the rest is refunded. **D28** | A8 |
| Q2 | What does project status `under_review` mean? | Only "draft sent to admin for AI band review" (FR-AI-04): draft -> under_review -> draft. The last approved stage moves the project in_progress -> completed directly. **D29** | A3 |
| Q3 | OCR and contact scanning of attachments/deliverables in Phase 1? | Extract text from PDF, DOCX, TXT, MD and run moderation layers 1 + 2; source zips virus-scan only; images virus-scan only and labelled "not text-scanned"; chat is text-only. OCR of images in Phase 2. **D30** | A3, A6, A7 |
| Q4 | Places "interests" format? | City (required) + 1-5 project categories (category ids). Shown once, editable in Settings. **D31** | A2, F2 |
| Q5 | Deadline as date or days for the 70% rule? | Client enters `deadline_days` (1-365); must be >= ceil(70% x AI days_min). Date shown for information only. **D32** | A3, F2 |
| Q6 | How are stage due dates calculated? | Agreed days split by stage percentage (largest remainder, each >= 1 day). Due date set when the stage becomes active = activation date + stage days, 23:59:59 Asia/Karachi; client review time never counts against the developer. Extension moves only that stage. **D33** | A5, A6, A8 |
| Q7 | Where does the admin AI rate table live? | Phase 1: `ai-service/data/rates.yaml`, changed by reviewed PR; admin screen read-only through new `GET /v1/rates` (contract change in B2). Editable table in Phase 2. **D34** | B2, A9, F5 |
| Q8 | Escrow and fee when a developer cancels before starting? | Project back to open; escrow + fee held; client may request a full refund (incl. fee) until a new bid is accepted. Next acceptance reuses escrow, fee recalculated; positive difference paid via 48 h window, negative refunded. Reminder email after 14 days; never auto-refunded. **D35** | A8 |
| Q9 | Client ends the assignment from late-day 8: full abandonment penalties? | **Yes**: system 1-star review, strike, reliability -20, counts toward 3 abandonments in 12 months; project reopens. **D36** | A8 |
| Q10 | Alert when the moderation queue waits > 2 h? | Yes. SLA 2 h counted only inside 09:00-23:00 Asia/Karachi; dashboard shows oldest item age; alert email on breach. **D37** | A9, A13 |
| C7 | Account status values? | `pending_verification, active, under_review, suspended, closed` (PDF blocked = suspended, pending = pending_verification). **D38** | A2 |

New open questions found during later tasks are added below this table as Q11, Q12, ...