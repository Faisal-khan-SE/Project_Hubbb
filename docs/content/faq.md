# FAQ — Platform Help Centre

> This file is the **single source of truth** for the platform FAQ.
> The B4 chatbot (`ai-service`) loads this file as its system prompt context.
> The F7 public FAQ page (`SCR-57`) renders it directly.
> Do not change numbers here without also updating `docs/decisions.md` and running the full test suite.

**Last updated:** 2026-11-15  
**Source decisions:** D1-D38, `docs/requirements.md` §3-§9

---

## For Clients

**What is this platform?**  
A marketplace where you can upload a software project, receive an AI-generated fair price and time estimate, compare bids from verified developers, pay securely into escrow, and receive your work in stages — without ever sharing your personal contact details with a developer.

**How does the AI estimate work?**  
When you upload a project, the AI reads your title, description, category, required skills, and any attached files. It returns a price range (PKR) and a time range (days), along with a complexity level (low / medium / high) and a list of features it found. This takes under 20 seconds. You cannot publish your project until the estimate is ready.

**What is the allowed budget range?**  
Your minimum budget must be at least **80% of the AI's lowest price estimate**, and your maximum budget must be no more than **130% of the AI's highest price estimate**. If you believe the estimate is wrong, you can request an admin review with a written reason (FR-AI-04).

**What is the allowed deadline?**  
Your project deadline must be at least **70% of the AI's minimum estimated days**. You enter the deadline as a number of days (1–365).

**Can I save a project without publishing?**  
Yes. Projects are saved as drafts and can be edited or published at any time before you receive bids.

**How does bidding work?**  
Once published, verified developers can place one bid per project. A bid contains a proposed price (PKR) and a proposed number of days. Both must be within the allowed band. Each bid may also include a short proposal text, which is checked by the moderation bot before you see it.

**Can I negotiate with a developer?**  
Yes. You can send a counter-offer to any bid. The developer can accept, counter, or reject. Each bid allows up to **3 negotiation rounds**. Each offer is valid for **48 hours**; if not answered within that time, the offer expires.

**What information do I see about a developer?**  
You see the developer's username, reliability score (0-100), average rating (1-5 stars), number of completed projects, proposed price and days, and their proposal text. You never see a developer's real name, email, or phone number.

**How do I accept a bid?**  
Click "Accept Bid" on the developer you want. The project moves to `awaiting_payment` status. You have **48 hours** to complete payment. If you do not pay within 48 hours, the acceptance lapses and the project returns to open bidding.

**What do I pay?**  
You pay the **agreed price plus a client service fee of 5% (minimum PKR 300)**. This fee covers payment gateway charges and is fixed at the moment you accept the bid — later fee changes do not affect your project.

**What payment methods are accepted?**  
JazzCash, Easypaisa, debit/credit card through a payment gateway, and bank transfer (confirmed manually by admin).

**Is my money safe?**  
Yes. Your payment goes into platform **escrow** (held securely). The developer is paid only after you approve each stage. If you cancel before any work is submitted, you receive a full refund including the client fee.

**How is the work delivered?**  
In three stages:

| Stage | What the developer submits | Your payment share |
|---|---|---|
| 1. SRS | Requirements, use cases, screen designs | 20% of agreed price |
| 2. Working Demo | Demo video + live demo link | 40% of agreed price |
| 3. Source Code & Handover | Full source code zip + setup guide | 40% of agreed price |

You can adjust these percentages with the developer before Stage 1 starts (they must total 100%).

**How do I approve a stage?**  
Once the developer submits a stage, you review the deliverable and either approve it or request a revision with written feedback. If you do not respond within **7 calendar days** of a submission, the stage is **automatically approved**.

**What if the developer is late?**  
There is a **24-hour grace period** after the stage due date. After that, a late penalty is deducted from the developer's stage payment and returned to you as a discount:

- Days 1–3 late (after grace): 2% of stage amount per day
- Day 4 onwards: 3% of stage amount per day
- Maximum penalty: 25% of the stage amount

Lateness is measured on the first submission. If you request a revision, the developer's revision time does not count against them.

**What if the developer is 8 or more days late?**  
From day 8 of lateness (after grace), you may choose to end the assignment. This is treated as abandonment by the developer (FR-LATE-02).

**What is a dispute?**  
If you disagree with a stage deliverable and cannot resolve it through revision requests, you can raise a dispute. The stage is frozen — no payment is made and auto-approve stops — until an admin reviews the evidence and decides: release the funds to the developer, refund them to you, or split the amount.

**Can I cancel my project?**  
Yes, but consequences depend on how many times you have cancelled in the past 12 months:

- **1st cancellation:** Warning only. Full refund including the client fee.
- **2nd cancellation:** You pay 5% of the agreed price to the developer as compensation.
- **3rd+ cancellation:** You pay 10% of the agreed price to the developer, receive a strike, and your account goes under review.

If work has already started, the developer is additionally entitled to 25% of the in-progress stage amount on the first offence (and the 5% / 10% compensation on later offences). Compensation is paid only from the escrow still held; you are never charged extra beyond what you already paid.

**What is the client fee refund policy?**  
The client fee is refundable **only** if the project is cancelled before Stage 1 is approved. After Stage 1 is approved, the fee is not refunded.

**What are strikes?**  
Strikes are issued for contact-sharing attempts (via the message moderation bot). First strike is a warning; second strike locks your messaging for 24 hours; third strike sends your account for admin review. Strikes expire after 6 months.

**How do I contact a developer?**  
You communicate through the platform's built-in messaging system. Each project has one conversation between you and the developer. All messages pass a moderation bot that blocks contact details (phone numbers, emails, social media handles, etc.).

**Can I leave a review?**  
Yes, after the project is completed, both you and the developer can leave a one-time rating (1–5 stars) and an optional comment (which is moderated).

**How do invoices work?**  
You receive an invoice for every payment you make. Invoices are numbered sequentially (INV-YYYY-xxxxxx) and can be downloaded as PDF from the Payment History page. Invoices are immutable once issued.

---

## For Developers

**How do I start bidding?**  
You must: (1) complete your developer profile, (2) submit KYC documents (CNIC + images) and have them approved by an admin, (3) add and have a payout account verified by admin. All three must be in place before you can place a bid.

**What is KYC?**  
Know Your Customer verification. You submit your CNIC number and front/back images of your CNIC card (selfie is optional). An admin reviews the documents and approves or rejects your submission with a reason. KYC data is visible only to platform admins and is never shared with clients.

**How does the reliability score work?**  
Your reliability score starts at 100 and can range from 0 to 100. It changes only through recorded events:
- Stage approved on time: **+2 points**
- Late stage delivery: **−5 points**
- Developer cancellation (2nd or later offence): **−10 points**
- Project abandonment: **−20 points**

Clients can see your score when comparing bids.

**What is the allowed bid range?**  
Your bid price must be between 80% and 130% of the AI estimate band for the project. Your proposed days must be at least 70% of the AI minimum days.

**Can I negotiate?**  
Yes. The client may send a counter-offer. You can accept, counter (with a new price + days + optional note up to 200 characters), or reject. Maximum 3 rounds per bid; each offer is valid for 48 hours.

**What is my commission?**  
10% of each stage payout, calculated after any late penalty is deducted. The commission rate is fixed at the time you agree the bid.

**When do I get paid?**  
After the client approves each stage, the system schedules a payout for the **next business day** (Monday–Friday, excluding Pakistan federal public holidays). Payouts are currently processed manually by platform admins — you will receive an email notification when each payout is marked as transferred.

**What is a payout receipt?**  
You receive a receipt (invoice) for every payout. Receipts are numbered PAY-YYYY-xxxxxx and can be downloaded as PDF from the Earnings page. They include tax lines (even when the tax rate is 0%).

**What if I cancel before starting?**  
Within 24 hours of a bid being accepted and before submitting anything:
- 1st time: Warning only.
- 2nd time and beyond: A strike and a 10-point reliability score drop.

After you submit your first deliverable, quitting counts as abandonment (see below).

**What is abandonment?**  
If you quit after submitting at least one deliverable, or if the client ends the assignment because you are 8 or more days late (after the grace period), it is counted as abandonment:
- Approved stages stay paid.
- A submitted but not-yet-approved stage goes to the client for a decision (approve and pay, or reject and refund) within 3 days.
- Unsubmitted stages are not paid.
- The system automatically posts a 1-star review marked "Project abandoned."
- You receive a strike and a 20-point reliability score drop.
- 3 abandonments in 12 months move your account to `under_review`.

**What happens to the project after I abandon?**  
The project reopens from the next uncompleted stage. The client can receive new bids. Completed stages and their deliverables remain visible to new bidders.

**Can I request a deadline extension?**  
Yes, before the stage due date. You can request up to 7 additional days with a written reason (which is moderated). If the client accepts, the due date moves and no penalty applies. You cannot request an extension after the due date has passed.

**Can I request a revision to be reconsidered?**  
No — only the client can request a revision. If you disagree with a revision request, you can use the dispute mechanism.

**How does messaging work?**  
Each project has one conversation between you and the client. Messaging opens from your first bid onward. All messages pass the moderation bot. If the bot holds a message for review, you see "Your message is being reviewed." If a message is rejected, you see the reason. Contact details (phone, email, social media) are blocked.

**What are strikes (developer)?**  
Same as for clients: 1st is a warning; 2nd locks your messaging for 24 hours; 3rd sends your account for admin review (you cannot place new bids but ongoing work continues).

---

## Payments & Escrow

**How is escrow managed?**  
The client's payment (agreed price + client fee) is held in a platform-controlled escrow. Stage payments are released only after the client approves a stage (or after 7 days auto-approve). The developer never has access to escrow funds until a stage is approved.

**What is auto-approve?**  
If the client does not approve or request a revision within **7 calendar days** of a stage submission, the stage is automatically approved and the payout is scheduled. Auto-approve does not apply if the stage is in dispute.

**Are there taxes?**  
Tax rules are stored as data and currently set to 0% inactive until confirmed by a tax advisor. Every invoice shows tax lines even when the rate is 0%. Once a tax advisor confirms the rates, an admin will activate them.

---

## Disputes & Cancellations

**How do I raise a dispute?**  
Either party can raise a dispute on the active stage. The stage is immediately frozen — no auto-approve, no payment — until an admin resolves it. Already-approved stages are not affected.

**How are disputes resolved?**  
An admin reviews the evidence and decides to: release funds to the developer, refund them to the client, or split the stage amount between both parties. The decision is final and creates the appropriate payment rows.

**Can I appeal an admin decision?**  
No formal appeal process is specified. Contact support via a support ticket.

---

## Platform Rules

**Is my contact information safe?**  
Yes. The platform never reveals your email address, phone number, or real name to the other party. Developers appear by username only. This is enforced at the API level.

**What if I see another user's contact details?**  
Please report it immediately via a support ticket. This is a critical bug.

**How does the moderation bot work?**  
Every message, bid proposal, offer note, extension reason, review comment, and deliverable note passes through a two-layer moderation system:
1. **Pattern matching** (Pakistani phone formats, email patterns, social media keywords, etc.)
2. **AI intent scoring** (detects subtle attempts to move communication off-platform)

Messages with a risk score below 0.3 are delivered instantly. Messages scoring 0.3-0.7 are held in an admin queue (you see "under review"). Messages above 0.7 are held and may result in a strike if an admin confirms contact-sharing intent.

