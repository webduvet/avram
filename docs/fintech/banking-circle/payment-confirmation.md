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
| `GET /api/v1/payments/singles/{payment-id}` or `.../status` | One payment by BC id |
| `GET /api/v1/payments/singles/transactionreference/{ref}` | If an own reference was stored at create |
| `GET /api/v1/reports/reconciliation-intraday-report` | Same-day list (time-range). After **19:00 CET** it rolls to the next business day |
| `GET /api/v1/reports/reconciliation-report` (or paged / async variants) | Historical. Includes `Processed` and `PendingProcessing` |

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
5. The settlement worker is the only ledger writer. Book only on BC `Processed` (payout confirmed) or `Rejected` / `Reversed` (do not treat as paid). Do **not** leave a hung “waiting for webhook” state across daily settlement.
6. First persistent webhook/crypto failure is a **same-day page**. Do not wait for the ~3h40m warning email or auto-deactivate.

## 6. Sweep of pending rows

Keep the webhook path when it exists. Always sweep the integrator’s **own pending rows** for when webhooks fail, are late, or were never subscribed.

Webhook = fast track (optional for MVP). Sweep = ultimate confirm (**required**). Extra B4B hook = opportunistic same job. **Same settlement-job payload.** Idempotent on `paymentId` + status so webhook, extra B4B hook, and sweep can all fire.

1. Select pending rows older than a short timeout. Each row already has the BC `paymentId` (from Oversight `B4BTMApproved` / `banking_circle_api_response.paymentId`, or from the BC create response).
2. Ultimate confirm: `GET /api/v1/payments/singles/{paymentId}/status` (or the full GET). Not B4B `GET /payments/{id}`.
3. If `Processed`, `Rejected`, or `Reversed`: enqueue the **same settlement job** the webhook would have. The settlement worker remains the only ledger writer (processed, or rejected + reverse the ledger entry).
4. If still `PendingProcessing` / `MissingFunding` / `Hold`: leave pending; **alert** if it ages past the settlement cut-off.
5. For many hung ids, prefer `GET /api/v1/reports/reconciliation-intraday-report` over N singles. After **19:00 CET** that report rolls to the next business day.
