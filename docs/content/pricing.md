# Pricing

> This file is the content source for the **SCR-56 Pricing page** (F7).
> All numbers on the public pricing page MUST be fetched from `GET /api/v1/public/config` at build/revalidate time.
> This file documents where each number comes from and shows the worked example.
> Do not type any fee or penalty number directly into frontend code — use the config endpoint (FR-CORE-05, A11).

**Last updated:** 2026-11-15  
**Source:** `docs/requirements.md` §3-§4, `docs/decisions.md` D1, D13, D23, D24, D25, BR-01..BR-10

---

## Fee Summary

| Fee | Amount | Notes |
|---|---|---|
| Client service fee | 5% of agreed price (minimum PKR 300) | Added on top of the agreed price at payment. Fixed at acceptance time (FR-PAY-03). |
| Developer commission | 10% of each stage payout | Deducted after any late penalty. Fixed at the time the bid is agreed. |
| Late penalty | 2% per day for days 1-3; 3%/day from day 4; cap 25% of stage amount | Returned to the client as a discount. Measured after a 24-hour grace period (D13). |

All numbers are defaults editable by the platform admin (stored in `fee_settings` and `penalty_rules`).  
Tax rates are currently 0% and inactive until confirmed by a tax advisor (D23). Every invoice shows tax lines even when 0%.

---

## Payment Window

| Event | Deadline |
|---|---|
| Client pays after accepting a bid | 48 hours (D4) |
| Client approves or requests revision after stage submission | 7 calendar days (auto-approve after this — D14) |
| Developer can submit a stage after the client's acceptance | Per stage due date, calculated from agreed days split by stage percentage |
| Payout after stage approval | Next business day (Mon-Fri, excluding Pakistan federal public holidays — D6, D12) |

---

## Payment Methods

| Method | Notes |
|---|---|
| JazzCash | Mobile wallet / card checkout (FR-PAY-02) |
| Easypaisa | Mobile wallet checkout (FR-PAY-02) |
| Debit / Credit Card | Processed through a payment gateway (FR-PAY-02) |
| Bank Transfer | Confirmed manually by admin (FR-PAY-02) |

---

## Delivery Stages (Default Percentages)

| Stage | Description | Payment Share |
|---|---|---|
| 1. SRS | Software Requirements Specification — requirements, use cases, screen designs | 20% of agreed price |
| 2. Working Demo | Demo video + live demo link | 40% of agreed price |
| 3. Source Code & Handover | Full source code, setup guide, final documents | 40% of agreed price |

The client and developer may adjust these percentages before Stage 1 starts; they must total 100% (FR-DEL-01).

---

## Late Penalty Table

| Days late after grace period | Penalty rate | Cumulative total at end of range |
|---|---|---|
| 0 (within 24-hour grace) | 0% | 0% |
| 1 to 3 | 2% of stage amount per day | 6% |
| 4 to 7 | 3% of stage amount per day | 18% |
| 8 and beyond | 3% of stage amount per day | up to 25% (cap) |

- Lateness is measured only from the **first submission** of the stage.
- Revision time (after a client revision request) does not count against the developer.
- The penalty goes back to the **client as a discount** — the platform does not keep it.
- Grace period: 24 hours after the due date (D13, BR-09, BR-10).

---

## Worked Example — PKR 100,000 Project

> **This exact example is used as a mandatory test fixture in task A8. Numbers must not be changed.**
> Source: `docs/requirements.md` §4.1

Agreed price: **PKR 100,000**  
Stage 2 (Working Demo) is delivered **2 days late** (after the 24-hour grace period).  
Stages 1 and 3 are on time.

| Step | Calculation | Amount (PKR) |
|---|---|---|
| Client pays at acceptance | 100,000 + 5% client service fee (5,000) | **105,000** |
| Stage 1 — SRS (20%) | Stage amount = 20,000. No late penalty. Commission = 10% × 20,000 = 2,000. | Developer receives **18,000** |
| Stage 2 — Working Demo (40%) | Stage amount = 40,000. Late penalty: 2 days × 2% = 4% of 40,000 = **1,600**. | Client refunded **1,600** |
| | Net stage after penalty = 38,400. Commission = 10% × 38,400 = 3,840. | Developer receives **34,560** |
| Stage 3 — Source Code (40%) | Stage amount = 40,000. No late penalty. Commission = 10% × 40,000 = 4,000. | Developer receives **36,000** |
| **Totals** | Developer: 18,000 + 34,560 + 36,000 = **88,560** | |
| | Platform: 5,000 (client fee) + 2,000 + 3,840 + 4,000 (commission) = **14,840** | |
| | Client net cost: 105,000 − 1,600 (penalty refund) = **103,400** | |
| | Check: 88,560 + 14,840 = **103,400** ✓ | |

> All amounts use **exact decimal arithmetic** (NUMERIC(12,2) in the database; BigDecimal in Java — D1). Never double or float.

---

## Invoices & Receipts

Every financial transaction generates an immutable invoice downloadable as PDF (D24, FR-INV-01):

| Series | For | Example number |
|---|---|---|
| INV-YYYY-xxxxxx | Client payments and fee refunds | INV-2026-000001 |
| PAY-YYYY-xxxxxx | Developer payouts | PAY-2026-000001 |

Invoice numbers are sequential and gap-free. Invoices show tax lines (even when rates are 0% — D23).

---

## Payout Mode

Payouts are currently **manual**: when a stage is approved and a payout is due, platform admins transfer the amount via bank transfer or mobile wallet and record the reference number. The developer is notified when the transfer is marked as complete (D25, FR-PAY-09).

This will switch to an automated disbursement API once a merchant agreement is in place. No action is required from developers — it is a configuration change.

