# How It Works

> Content source for **SCR-55** (How It Works public page, F7) and the platform onboarding flow.
> Numbers in this file must match `docs/decisions.md` and `docs/content/pricing.md` exactly.
> Do not add features that are not in `docs/requirements.md` or `docs/decisions.md`.

**Last updated:** 2026-11-15  
**Source:** `docs/requirements.md` §13 (Key flows), `docs/decisions.md` D1-D38

---

## Overview

This platform connects **clients** who need software built with **verified developers** who can build it.

Three principles protect everyone:

1. **No direct contact** — neither side ever sees the other's phone number, email, or real name. Developers appear by username only (FR-CORE-01, D11).
2. **No direct payment** — clients pay into platform escrow; developers are paid only after work is approved (FR-CORE-02).
3. **Fair price** — an AI analyses every project and sets a price and time band. All bids must stay within this band (FR-CORE-03).

---

## Client Journey

### Step 1 — Sign up and verify your email

Create an account with your name, email, and password. Choose the **Client** role. Accept the platform's Terms of Service, Privacy Policy, and Refund & Cancellation Policy.

You will receive a 6-digit verification code by email. Enter it to activate your account (FR-AUTH-01, D20).

After your first login, a quick pop-up asks for your city and the types of projects you are interested in. This helps personalise your experience (FR-AUTH-04, D31).

### Step 2 — Upload your project

Fill in the project title, description, category, required skills, your budget range (PKR), and your deadline (in days). You can attach supporting files such as an SRS document or wireframes (FR-PRJ-01).

Save as a draft first. When ready, click **Request Estimate**.

### Step 3 — Receive the AI estimate

The platform's AI analyses your project and returns:
- A fair **price range** (PKR minimum and maximum)
- A fair **time range** (days minimum and maximum)
- A **complexity level** (low / medium / high)
- A list of **features** it found in your description

This takes under 20 seconds. While you wait, your project stays in `estimate_pending` status (FR-AI-01, NFR-PRF-03).

Your budget must be between **80% and 130% of the AI estimate band**, and your deadline must be at least **70% of the AI minimum days** (FR-AI-02, FR-AI-03). If you believe the estimate is wrong, you can request an admin to review and override it (FR-AI-04).

### Step 4 — Publish your project

Once the estimate looks right and your budget and deadline are within the allowed band, click **Publish**. Your project is now visible to all verified developers.

### Step 5 — Compare bids

Verified developers place bids with a proposed price, proposed days, and a short proposal note. You can:
- Compare bids side by side, seeing each developer's reliability score (0-100), average rating, and completed projects count — but never their real name, email, or phone (FR-BID-06).
- Short-list developers you are interested in.
- Reject bids you do not want.
- **Counter-offer** any bid with a new price and days (FR-BID-02). You can negotiate simultaneously with multiple developers. Each negotiation allows up to 3 rounds, and each offer is valid for 48 hours.

### Step 6 — Accept a bid and pay

When you are ready, accept one bid. The project moves to `awaiting_payment`. You have **48 hours** to complete payment (FR-PAY-01, D4).

You pay the **agreed price plus a client service fee of 5% (minimum PKR 300)**. This total goes into secure escrow. No money reaches the developer yet.

Supported payment methods: JazzCash, Easypaisa, card, or bank transfer (FR-PAY-02).

You will receive an invoice for this payment (D24, FR-INV-01).

### Step 7 — Work is delivered in three stages

Once payment is confirmed, the developer begins work. Delivery happens in three stages:

| Stage | What you receive | Your share |
|---|---|---|
| 1. SRS | Software Requirements Specification — requirements, use cases, screen designs | 20% |
| 2. Working Demo | Demo video + live demo link (no source code yet) | 40% |
| 3. Source Code & Handover | Full source code, setup guide, final documents | 40% |

You and the developer may agree on different percentages before Stage 1 starts; they must total 100% (FR-DEL-01).

### Step 8 — Approve each stage

After the developer submits a stage, you review the deliverable. You can:
- **Approve** — the payout for this stage is scheduled for the next business day.
- **Request revision** — provide written feedback; the developer resubmits.

If you do not respond within **7 calendar days**, the stage is **automatically approved** (FR-DEL-05, D14).

Late deliveries result in a **penalty deducted from the developer's payment and returned to you as a discount** — not kept by the platform (D13, BR-10). There is a 24-hour grace period before penalties start.

### Step 9 — Source code is released on final approval

The source code download is locked until Stage 3 is approved. Once you approve Stage 3, the source code becomes available to download (FR-DEL-06).

### Step 10 — Leave a review

After the project is completed, you can leave a one-time rating (1-5 stars) and an optional comment for the developer. The developer can also rate you. Comments are moderated (FR-REV-01).

---

## Developer Journey

### Step 1 — Sign up and verify your email

Create an account with your name, email, and password. Choose the **Developer** role. Accept the Terms of Service, Privacy Policy, Refund & Cancellation Policy, and the Developer Agreement (FR-AUTH-01, D27).

Verify your email with the OTP code sent to your inbox (D20).

### Step 2 — Complete your profile

Fill in:
- A unique **username** (this is what clients see — not your real name, D11)
- Headline and bio
- Years of experience and portfolio URL
- Skills (choose from the platform's list)

A complete profile is required before you can bid (FR-AUTH-03).

### Step 3 — Add a payout account

Add your bank account, JazzCash, or Easypaisa account where you want to receive payments. The account number is stored encrypted; only the last 4 digits are shown in the UI (D11, NFR-SEC-04). An admin must verify the account before you can bid (FR-PAY-04).

### Step 4 — Complete KYC

Submit your CNIC number and front and back images of your CNIC card (selfie is optional). An admin reviews the documents and approves or rejects your submission. KYC data is visible only to platform admins and is never shared with clients (D26, FR-KYC-01).

Once KYC is approved and your payout account is verified, your account can be verified by an admin.

### Step 5 — Browse open projects

Browse projects posted by clients. Filter by category, skills, budget range, and deadline. Each project shows the AI-estimated price and time band so you can see what a fair bid looks like (FR-AI-05).

### Step 6 — Place a bid

Submit one bid per project with:
- A proposed **price (PKR)** — must be within the allowed band (80%-130% of AI estimate)
- A proposed **number of days** — must be at least 70% of the AI minimum days
- A **proposal text** describing your approach

The proposal is checked by the moderation bot before the client sees it (D8, FR-BID-01).

### Step 7 — Negotiate if the client counters

If the client sends a counter-offer, you can accept, counter back (up to 3 rounds total), or reject. Each offer is valid for 48 hours. Notes in counter-offers are limited to 200 characters and are moderated (BR-07, FR-BID-02).

### Step 8 — Wait for acceptance and payment

If the client accepts your bid, you will be notified. The project moves to `awaiting_payment` — you do not start work yet. Once the client pays (they have 48 hours), you will receive a notification that the project is `in_progress` and work can begin.

### Step 9 — Deliver work in three stages

Work through the three stages:

**Stage 1 — SRS (20%)**  
Deliver a full Software Requirements Specification: requirements, use cases, and screen designs as agreed in the project brief.

**Stage 2 — Working Demo (40%)**  
Deliver a demo video and a live demo link. The source code is not submitted yet.

**Stage 3 — Source Code & Handover (40%)**  
Deliver the full source code zip and a setup guide. The source code download is locked for the client until this stage is approved.

For each stage, upload your deliverable through the platform. Files are virus-scanned and document text is checked for contact details before the client can download anything (D30, FR-DEL-03).

### Step 10 — Request extensions if needed

Before a stage's due date, you may request up to 7 extra days with a written reason. If the client accepts, the due date moves and no penalty applies. If the client rejects or you do not ask in time, the original due date stands (FR-LATE-01).

### Step 11 — Get paid stage by stage

After the client approves a stage (or after 7 days auto-approve), the platform schedules your payout for the **next business day** (Monday–Friday, excluding Pakistan federal public holidays). Your net payout is the stage amount minus any late penalty minus the 10% platform commission (BR-02, BR-03, D13).

You receive a downloadable payout receipt (PAY-YYYY-xxxxxx series) for each payout (D24, FR-INV-01).

### Step 12 — Leave a review

After the project is completed, leave a one-time rating (1-5 stars) and an optional comment for the client (FR-REV-01).

---

## Escrow Protection — How Your Money Is Safe

```
Client pays agreed price + 5% fee
            ↓
      Platform Escrow
            ↓
  Stage 1 approved → Developer receives 20% − commission
  Stage 2 approved → Developer receives 40% − commission (minus any penalty)
  Stage 3 approved → Developer receives 40% − commission
            ↓
   Any late penalty → Returned to client as a discount
   Platform keeps: client fee + all commissions
```

No money moves directly between client and developer. Every movement is recorded in the payments ledger (FR-PAY-06, D1, D18).

---

## Messaging Rules

All communication goes through the platform's built-in messaging. Contact details (phone numbers, email addresses, social media handles, payment apps) are blocked by the moderation bot. This protects both parties and ensures all agreements stay on the platform (FR-CORE-01, FR-MSG-01, D8).

---

## What Happens if Something Goes Wrong?

| Situation | What happens |
|---|---|
| Developer is late | Penalty accrues and is returned to client. Client can end the assignment from day 8 of lateness. |
| Developer abandons the project | Approved stages stay paid. The project reopens for new bids. Developer receives a strike and reliability score drop. |
| Client cancels | Compensation is paid to the developer (amounts depend on offence history). Full refund to client on first cancellation. |
| Dispute | Either party can raise a dispute. The stage is frozen until an admin decides: release, refund, or split. |
| Contact-sharing attempt | The moderation bot holds the message. Admin reviews. Confirmed attempts result in a strike. |

