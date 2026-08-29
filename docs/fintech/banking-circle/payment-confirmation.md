# Payment confirmation

How to confirm a Banking Circle payment is **Processed** or **Rejected**. Webhooks are the fast track. If a webhook fails or takes too long, a **sweep of the integrator’s own pending rows** (poll / recon) is the ultimate path. A virtual ledger must **not** hang on a missing POST.

Integrator flow: a Connect webhook with payment id + status (`Processed` / `Rejected`) enqueues a **settlement job**. A settlement worker moves a pending virtual-ledger txn to processed, or on `Rejected` moves it to rejected and reverses the ledger entry. The sweep enqueues **the same job**. An undocumented extra B4B Oversight callback may also enqueue it (opportunistic). The settlement worker is the only ledger writer.

**MVP:** Banking Circle is **not** cut. The BC webhook subscription **may** be skipped. The **BC read must not** be skipped (`GET /api/v1/payments/singles/{paymentId}/status` or recon).

Related: [Webhooks](webhooks/index.md) · [ACK vs booking](webhooks/faq.md#ack-vs-booking) · [B4B Oversight payment tracking](../b4b-payments/oversight-payment-tracking.md)

Sources: [Payment status](https://docs.bankingcircleconnect.com/docs/payment-status) · [Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments) · [Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report) · [Webhook retry strategy](https://docs.bankingcircleconnect.com/docs/webhook-retry-strategy) · [Reconciliation using webhooks](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks)

## 1. Fast track — Connect webhooks

Subscribe to payment events, including `OutgoingPaymentProcessed`, `OutgoingPaymentRejected`, `OutgoingPaymentBooked`, `Reversed`, `PaymentStatus`, and the incoming equivalents. See [Setup webhooks](webhooks/setup-webhooks.md).

ACK with `2xx` quickly so the subscription stays alive. Book the virtual ledger **only** on AES-GCM success, or after poll/recon confirms. Details: [ACK vs booking](webhooks/faq.md#ack-vs-booking).

On a verified `Processed` / `Rejected` webhook, enqueue the settlement job. Do not write the ledger in the webhook handler.

A missed or failed webhook does **not** reverse the payment. `Reversed` means the **scheme** rejected it. Once `2xx`, that notification is not resent. Non-`2xx` retries, then can deactivate the subscription; new events in the inactive window are not queued.

MVP may omit this subscription. Ultimate confirm is still the sweep / BC GET.

## 2. Ultimate confirmation — poll / reports

Use this when the webhook is silent, late, untrusted, or **not subscribed**. The **sweep** below is how this is applied to hung pending rows. This path is **required** even if the BC webhook is skipped.

| Call | Use |
|---|---|
| `GET /api/v1/payments/singles/{payment-id}` or `.../status` | One payment by BC id. Fallback when a pending row is missing from the intraday report (`Rejected` is not documented as in-scope there) |
| `GET /api/v1/payments/singles/transactionreference/{ref}` | If an own reference was stored at create |
| `GET /api/v1/reports/reconciliation-intraday-report` | Same-day list (time-range). **19:00 CET is a creation cutoff** (payments **created** after 19:00 CET appear next business day), not report availability. Morning issues are on a **midday** pull |
| `GET /api/v1/reports/reconciliation-report` (or paged / async variants) | Historical. Includes `Processed` and `PendingProcessing` |

The intraday report includes **only** `processed` and `pendingProcessing`. **Presence ≠ `Processed`.** Use `processedTimestamp` (optional; request it) or leave `pendingProcessing` pending.

Match recon lines on `paymentId`, `userReferenceNumber`, or remittance / end-to-end (`paymentDetails1`). camt.053 / camt.052 are the ISO form of the same books.

B4B `GET /payments/{id}` is **regulatory-phase only**. It is not ultimate confirm.

## 3. What “confirmed” means

Banking Circle has **no `Settled` status**.

| Status / event | Meaning for a virtual ledger |
|---|---|
| `Processed` | Final for a successful outgoing: passed validation/AML, **sent to the recipient**. Usual “payment went out” signal |
| `Rejected` | Final; not sent (invalid fields, or `MissingFunding` for 2 business days) |
| `Reversed` | Scheme rejected **after** `Processed`. Rare. Final |
| Returned | **Not a status.** Original stays `Processed`; a new incoming appears (often remittance `RETURN OF PAYMENT` / `return:true`) |
| `Booked` | Accounting event on the account (`OutgoingPaymentBooked` / `IncomingPaymentBooked`). Independent of `Processed`; order not guaranteed |
| `PendingProcessing` / `MissingFunding` / `Hold` | **Not confirmed.** Keep polling |

## 4. If the payment is created via B4B Oversight

Oversight is **pre-settlement only**. Published docs: six statuses; `B4BTMApproved` / `B4BFailed` are terminal; webhook stops at handoff; settlement updates come from BC.

At `B4BTMApproved`, store `external_ref` → B4B id → `banking_circle_api_response.paymentId`. That BC id is what to poll, sweep, and match on recon. **Do not book from `B4BTMApproved`.** It only means the BC id exists and can be swept.

**Observed:** a later Oversight callback after `B4BTMApproved`, when BC has `Processed` the payment, has been seen in the wild. Published docs do not list it as a product. One ambiguous sentence (rail appears on the payment callback once processed) is **not enough** to treat it as supported. Ask B4B **in writing** before treating the extra hook as a product.

**Decision (MVP):** do **not** cut Banking Circle. The extra B4B hook **may** enqueue the **same settlement job** if `banking_circle_api_response.status` is `Processed` or `Rejected` (opportunistic, unsigned, undocumented). Sweep / ultimate confirm remains **BC** `GET /api/v1/payments/singles/{paymentId}/status` or recon.

Full note: [Oversight payment tracking](../b4b-payments/oversight-payment-tracking.md).

## 5. Operational outline

1. Create the payment (B4B or BC). Persist own `external_ref` / debtor reference and, at handoff, the BC `paymentId`.
2. If a BC webhook is subscribed and arrives with GCM verified, enqueue the settlement job (`Processed` / `Rejected`).
3. An extra B4B hook after handoff **may** enqueue the same job if `banking_circle_api_response.status` is `Processed` / `Rejected`. Still not ultimate confirm.
4. If nothing arrives, or decrypt-fails, or the timed wait expires: the **sweep** runs (next section). MVP **must** have this even with no BC webhook subscription.
5. The settlement worker is the only ledger writer. Book only on BC `Processed` (payout confirmed) or `Rejected` / `Reversed` (do not treat as paid). Do **not** leave a hung “waiting for webhook” state across daily settlement. Morning issues must be sendable by **16:00 local** — do not wait for 19:00 CET.
6. First persistent webhook/crypto failure is a **same-day page**. Do not wait for the ~3h40m warning email or auto-deactivate.

## 6. Sweep of pending rows

Sweep calls (official Connect reference slugs):

| Call | When | Official |
|---|---|---|
| `GET /api/v1/reports/reconciliation-intraday-report` | First sweep (bulk, midday) | [API](https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-reports-reconciliation-intraday-report). Described on [reconciliation-report](https://docs.bankingcircleconnect.com/docs/reconciliation-report) (processed + pendingProcessing; 19:00 CET creation cutoff) |
| `GET /api/v1/payments/singles/{payment-id}/status` | Leftovers / Rejected / not on the report | [API](https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-payments-singles-payment-id-status). Status meanings: [payment-status](https://docs.bankingcircleconnect.com/docs/payment-status) |

Keep the webhook path when it exists. Always sweep the integrator’s **own pending rows** for when webhooks fail, are late, or were never subscribed.

**Cadence (local time).** Issue window is about **08:00–11:00**. Must be able to send by **16:00**. Cannot wait for 19:00 CET.

| When | What |
|---|---|
| Fast path, anytime | Webhook `Processed` / `Rejected` → same settlement job |
| ~**12:00**, or **one hour after last issue** | First sweep: `GET /api/v1/reports/reconciliation-intraday-report` |
| Before **16:00** | Second sweep of leftovers (slow rails may still be `pendingProcessing` at +60 minutes) |

Webhook = fast track (optional for MVP). Sweep = ultimate confirm (**required**). Extra B4B hook = opportunistic same job. **Same settlement-job payload.** The worker is **idempotent** on `paymentId` + status: already settled or rejected from webhooks are no-ops.

1. First sweep pulls the **intraday report**, not N singles. Morning issues are on that midday pull. **19:00 CET** on that report is a **creation cutoff** (payments created after 19:00 CET appear next business day), not when the report becomes available.
2. The report includes only `processed` and `pendingProcessing`. **Presence ≠ `Processed`.** Use `processedTimestamp` (optional; request it) or leave `pendingProcessing` pending. For each `paymentId` that is processed, enqueue the **same settlement job**.
3. `Rejected` is **not documented as in-scope** on the report. Pending rows **missing** from the report → `GET /api/v1/payments/singles/{paymentId}/status`, then the same job.
4. If still `PendingProcessing` / `MissingFunding` / `Hold`: leave pending. Slow rails may still be `pendingProcessing` at +60 minutes; the **second sweep before 16:00** catches leftovers. Alert if it ages past the send-by-16:00 cut-off.

**Sources / references** (Connect docs are login-walled; B4B ReadMe is password-gated — do not store the password.)

Banking Circle Connect:

- [Intraday recon guide + fields](https://docs.bankingcircleconnect.com/docs/reconciliation-report)
- [Intraday report API](https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-reports-reconciliation-intraday-report)
- [Standard recon API](https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-reports-reconciliation-report)
- [GET payment](https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-payments-singles-payment-id)
- [GET payment status](https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-payments-singles-payment-id-status)
- [GET by transaction reference](https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-payments-singles-transactionreference-transactionreference)
- [List payments](https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-payments-singles)
- [Payment status meanings](https://docs.bankingcircleconnect.com/docs/payment-status)
- [Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments) (Processed / Rejected / Reversed)
- [Payment lifecycle](https://docs.bankingcircleconnect.com/docs/payment-lifecycle)
- [Webhooks](https://docs.bankingcircleconnect.com/docs/webhooks)
- [Retry / deactivate](https://docs.bankingcircleconnect.com/docs/webhook-retry-strategy)
- [Recon via webhooks](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks) (do not use webhooks as sole recon)

B4B Oversight:

- [Beneficiaries and Payments](https://b4bpayments.readme.io/docs/beneficiaries-and-payments) (handoff + dual-API)
- [Payment lifecycle / six statuses](https://b4bpayments.readme.io/docs/beneficiaries-and-payments#payment-lifecycle-and-status-values) (`B4BTMApproved` terminal)
- [Payment webhook callbacks](https://b4bpayments.readme.io/docs/beneficiaries-and-payments#payment-webhook-callbacks) (webhook stops at the handoff)
- [After handoff → BC](https://b4bpayments.readme.io/docs/beneficiaries-and-payments#after-handoff-integrating-with-banking-circles-webhook)
- [GET /payments and GET /payments/{id}](https://b4bpayments.readme.io/docs/beneficiaries-and-payments#listing-and-retrieving-payments)

Internal: [Oversight payment tracking](../b4b-payments/oversight-payment-tracking.md) · [Webhooks](webhooks/index.md) · [ACK vs booking](webhooks/faq.md#ack-vs-booking)
