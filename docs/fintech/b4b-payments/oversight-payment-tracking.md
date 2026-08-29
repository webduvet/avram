# Oversight payment tracking

How a payout moves from **B4B Oversight** (regulatory / pre-settlement) to **Banking Circle** (settlement). Integrators are expected to use **both** APIs. Source: B4B Payments ReadMe, Beneficiaries and Payments (Oversight guide) on `b4bpayments.readme.io`.

Banking Circle settlement facts: [Webhooks FAQ](../banking-circle/webhooks/faq.md) · [Setup webhooks](../banking-circle/webhooks/setup-webhooks.md).

## Oversight stops at handoff

Oversight handles regulatory / pre-settlement **only**. Terminal Oversight statuses:

| Status | Meaning |
|---|---|
| `B4BTMApproved` | Handed to Banking Circle, and BC **accepted** |
| `B4BFailed` | Failed in Oversight; not handed off |

The Oversight webhook **stops at handoff**. Settlement updates come from Banking Circle, not B4B.

At `B4BTMApproved`, `banking_circle_api_response.paymentId` is the Banking Circle id for `GET` payments, recon, and BC webhooks. Store `external_ref` → B4B id → BC `paymentId` at this point.

A later B4B callback claiming BC settlement is **undocumented**. Do not treat it as supported.

## What each poll means

**B4B** (does **not** mean settled on the safeguarding account):

- `GET /payments/{id}`
- `GET /payments?external_ref=` (paginated list)

**Banking Circle:**

- `GET /api/v1/payments/singles/{id}` or `.../status`
- `GET` reconciliation-intraday-report (same day; after 19:00 CET it rolls)
- `GET` reconciliation-report

BC has **no `Settled` status**. `Processed` = sent to the recipient. `Booked` = account event. `Processed` and `Booked` may arrive in either order.

## Hung virtual-ledger payouts

1. Persist `external_ref` → B4B id → BC `paymentId` at `B4BTMApproved`.
2. Poll B4B until handoff (or `B4BFailed`).
3. Poll BC (or wait for BC webhooks + recon) after handoff.
4. **Book on the virtual ledger only on BC `Processed` or a recon line** — not on Oversight approval, not on a B4B poll that still looks pre-settlement, not on an undocumented “BC settled” B4B callback.

Webhooks remain a hint. Drift control is the reconciliation report. ACK vs booking on the BC POST: [FAQ](../banking-circle/webhooks/faq.md#ack-vs-booking).
