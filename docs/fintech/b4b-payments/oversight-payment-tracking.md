# Oversight payment tracking

How a payout moves from **B4B Oversight** (regulatory / pre-settlement) to **Banking Circle** (settlement). Integrators are expected to use **both** APIs. Source: B4B Payments ReadMe, Beneficiaries and Payments (Oversight guide) on `b4bpayments.readme.io`.

How to confirm the BC payment after handoff: [Payment confirmation](../banking-circle/payment-confirmation.md). ACK vs booking: [Webhooks FAQ](../banking-circle/webhooks/faq.md#ack-vs-booking). Setup: [Setup webhooks](../banking-circle/webhooks/setup-webhooks.md).

## Oversight stops at handoff (documented)

Oversight handles regulatory / pre-settlement **only**. Published Oversight docs: **six statuses**; terminal statuses:

| Status | Meaning |
|---|---|
| `B4BTMApproved` | Handed to Banking Circle, and BC **accepted** |
| `B4BFailed` | Failed in Oversight; not handed off |

Documented: the Oversight webhook **stops at handoff**; settlement updates come from Banking Circle, not B4B. One ambiguous sentence says the rail appears on the payment callback once processed — **not enough** to treat post-handoff callbacks as a supported product.

At `B4BTMApproved`, `banking_circle_api_response.paymentId` is the Banking Circle id for `GET` payments, recon, and BC webhooks. Store `external_ref` → B4B id → BC `paymentId` at this point. `B4BTMApproved` only means the BC id exists and can be swept.

## Does Oversight create a Connect webhook subscription?

**No published sentence says payment-create creates or replaces a Banking Circle Connect subscription.** Oversight docs require the **integrator** to subscribe to BC webhooks. Settlement updates come from BC, not Oversight. The Oversight webhook stops at `B4BTMApproved`. The create-payment OpenAPI callback is only `POST` to the client `callback_url`.

The string `push_notifications` / `banking_circle/push_notifications` does **not** appear in B4B guides or the 94 reference slugs checked. `staging.b4bpayments.com` in the hub is the On-Platform `/services/json` server, not that path. An observed Connect endpoint `https://staging.b4bpayments.com/banking_circle/push_notifications` is **not documented** as an Oversight auto-subscribe.

Connect: the integrator `POST`s a subscription and **sets the URL**; multiple URLs are allowed; identical URLs are forbidden; `DELETE` and portal Remove drop a subscription; `PUT` can change the endpoint. No Connect page mentions B4B auto-subscribe.

Treat Connect subscriptions as **integrator-owned**. See [Setup webhooks](../banking-circle/webhooks/setup-webhooks.md), [Webhooks FAQ](../banking-circle/webhooks/faq.md#does-oversight-auto-subscribe-a-connect-webhook), and [Questions and undocumented features](questions-and-undocumented-features.md).

Sources: [Beneficiaries and Payments](https://b4bpayments.readme.io/docs/beneficiaries-and-payments) · [Payment webhook callbacks](https://b4bpayments.readme.io/docs/beneficiaries-and-payments#payment-webhook-callbacks) · [After handoff → BC](https://b4bpayments.readme.io/docs/beneficiaries-and-payments#after-handoff-integrating-with-banking-circles-webhook) · [Connect webhooks](https://docs.bankingcircleconnect.com/docs/webhooks) · [Webhook subscriptions](https://docs.bankingcircleconnect.com/docs/webhook-subscriptions)

## Observed vs documented

A **later Oversight callback after `B4BTMApproved`**, when BC has `Processed` the payment, has been seen in the wild. That hook is **opportunistic, unsigned, and undocumented**. Ask B4B **in writing** before treating it as a product.

**Decision (MVP):** do **not** cut Banking Circle. The extra B4B hook **may** enqueue the **same settlement job** if `banking_circle_api_response.status` is `Processed` or `Rejected`. Ultimate confirm remains a **BC read**: `GET /api/v1/payments/singles/{paymentId}/status` or recon. MVP **may skip the BC webhook subscription**. It **must not skip the BC read**.

## What each poll means

**B4B** — regulatory-phase only (does **not** mean settled on the safeguarding account):

- `GET /payments/{id}`
- `GET /payments?external_ref=` (paginated list)

**Banking Circle** — settlement / ultimate confirm:

- `GET /api/v1/payments/singles/{id}` or `.../status`
- `GET` reconciliation-intraday-report (same day; after 19:00 CET it rolls)
- `GET` reconciliation-report

BC has **no `Settled` status**. `Processed` = sent to the recipient. `Booked` = account event. `Processed` and `Booked` may arrive in either order.

## Hung virtual-ledger payouts

1. Persist `external_ref` → B4B id → BC `paymentId` at `B4BTMApproved`.
2. Poll B4B until handoff (or `B4BFailed`).
3. After handoff: optional extra B4B hook may enqueue the settlement job if `banking_circle_api_response.status` is `Processed` / `Rejected` (undocumented). **Sweep / ultimate confirm is still BC GET or recon.**
4. Book on the virtual ledger only via the settlement worker (BC `Processed` or recon line / `Rejected` + reverse). Do not treat Oversight approval, B4B `GET /payments/{id}`, or an undocumented “BC settled” callback as sufficient on their own.

Webhooks remain a hint. Drift control is the reconciliation report.
