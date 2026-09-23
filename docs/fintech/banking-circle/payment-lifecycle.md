# Payment lifecycle (outgoing)

Statuses and transitions for **outgoing** Banking Circle Connect payments. Docs describe the common path; they are **not** a full formal state machine for every race or concurrent workflow.

Related: [Payment confirmation](payment-confirmation.md) · [Midday recon sweep](midday-recon-sweep.md) · [Reverse vs Return](reverse-vs-return.md) · [Documentation extract — payments](documentation-extract/payments.md) · [Webhooks](webhooks/index.md)

## After initiation

After a successful initiate, track status via payment status endpoints and/or webhooks ([Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments)).

Typical next states:

| Status | Meaning |
| --- | --- |
| `PendingProcessing` | Waiting to be processed or in progress |
| `PendingApproval` | Waiting for another user (if approval workflows apply) |
| SCA variants (`ScaPending`, `ScaExpired`, `ScaFailed`, `ScaDeclined`, and cancellable `PendingSca` naming in cancel rules) | Strong Customer Authentication path when enabled |

Approval- and SCA-related statuses depend on company setup and user permissions ([Payment status](https://docs.bankingcircleconnect.com/docs/payment-status)).

## Success

| Transition | Final? |
| --- | --- |
| → `Processed` | **Yes** for a normal successful send. Passed validation/AML and sent to the recipient. |

## Insufficient funds

Quoted from [Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments):

> `MissingFunding`: Payment has not been processed due to insufficient funds in the debtor account. It will be automatically processed once the account is funded within **2 days**. If the account remains unfunded for **2 days**, the payment will be rejected

Same window in [Payment status](https://docs.bankingcircleconnect.com/docs/payment-status) (“two days”) and rejection-report OpenAPI (“included in the report for **2 days**”). Webhook event text says “**2 business days**” ([Webhooks](https://docs.bankingcircleconnect.com/docs/webhooks)) — prefer the outgoing-payments / payment-status wording unless Connect clarifies.

| Path | Result |
| --- | --- |
| Funded within the **2-day** window | Auto → `Processed` |
| Still unfunded after **2 days** | → `Rejected` (**final**) |

## Invalid data / other reject

→ `Rejected` (**final**). May also follow prolonged `MissingFunding`, or invalid field values ([Payment status](https://docs.bankingcircleconnect.com/docs/payment-status)). Rejection reason is on payment details (`errors`); see Connect rejection-reasons docs.

## Company cancel

→ `Cancelled` (**final**). Cancel is only available for certain pre-process statuses (e.g. `PendingProcessing`, `PendingApproval`, SCA / declined-by-approver states). **Cannot** cancel `Processed`, `Rejected`, `Cancelled`, `Reversed`, or `MissingFunding` ([Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments)). Endpoint: `PUT /api/v1/payments/singles/{payment-id}/cancel`.

## Scheme fail after Processed

Rare: payment marked `Processed` then rejected by the scheme → `Reversed` (**final**), **same** `paymentId` ([Payment status](https://docs.bankingcircleconnect.com/docs/payment-status), [Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments)).

## Returned is not a payment status

**Returned** is **not** a Connect payment status.

- Original outgoing stays **`Processed`**.
- Money back appears as a **separate incoming** payment (`IncomingPaymentProcessed` / recon row), typically with `return: true`, and a **different** `paymentId`.
- Link via remittance / original payment reference ([Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments); [Practical guide — returns](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks); recon field `return` on [Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report)).

See also: [Reverse vs Return](reverse-vs-return.md) for scheme reverses versus beneficiary returns.

## How to track

| Mechanism | Role |
| --- | --- |
| Webhooks | Real-time events (see below). Docs: do **not** use webhooks as sole reconciliation ([Webhooks](https://docs.bankingcircleconnect.com/docs/webhooks), [Practical guide](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks)). |
| `GET /api/v1/payments/singles/{payment-id}/status` | Status only; efficient poll ([OpenAPI](https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-payments-singles-payment-id-status)) |
| `GET /api/v1/payments/singles/{payment-id}` | Full details including history / rejection `errors` |
| `GET /api/v1/payments/singles` | Paginated list; filter by status, date range, etc. ([Payment lifecycle](https://docs.bankingcircleconnect.com/docs/payment-lifecycle)) |
| Reconciliation / rejection reports | Bulk completeness — see [Midday recon sweep](midday-recon-sweep.md) |

## Outgoing webhook events

From [Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments) and [Webhooks](https://docs.bankingcircleconnect.com/docs/webhooks) (spellings verified):

| Event | When |
| --- | --- |
| `OutgoingPaymentProcessed` | Successfully processed → status `Processed` |
| `MissingFunding` | Insufficient funds → status `MissingFunding` |
| `OutgoingPaymentRejected` | Rejected → status `Rejected` |
| `Reversed` | Scheme reverse → status `Reversed` |

Also relevant for ledger timing (not status substitutes): `OutgoingPaymentBooked`, and broader `PaymentStatus`. Incoming returns: `IncomingPaymentProcessed` with `return: true`.

## Sources

- https://docs.bankingcircleconnect.com/docs/outgoing-payments
- https://docs.bankingcircleconnect.com/docs/payment-status
- https://docs.bankingcircleconnect.com/docs/payment-lifecycle
- https://docs.bankingcircleconnect.com/docs/webhooks
- https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks
- https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-payments-singles-payment-id-status
- https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-payments-singles
