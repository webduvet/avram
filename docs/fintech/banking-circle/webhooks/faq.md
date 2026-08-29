# FAQ

## If Banking Circle fails to deliver a webhook, is the payment reversed?

No. Webhooks are notifications. Payment status `Reversed` means the **scheme rejected the payment**, not that a webhook failed. Failed delivery retries, then deactivates the subscription; it does **not** unwind the booking. Reconciliation reports and `GET` payments are the source of truth.

Sources: [Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments) · [Webhook retry strategy](https://docs.bankingcircleconnect.com/docs/webhook-retry-strategy) · [Reconciliation using webhooks](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks)

## ACK vs booking

Do **not** return non-`2xx` to Banking Circle for a persistent header or checksum misconfig. That retries, then deactivates the subscription; new events in the hole are not queued. Once `2xx`, that notification cannot be resent.

Enforce correctness on the **virtual ledger**, not on the HTTP status:

- ACK (`2xx`) means “received the POST.”
- Book only if AES-GCM decrypt (tag) succeeds, **or** if the reconciliation report / `GET` payments confirms the payment.
- Do **not** book on “headers look wrong but `paymentId` might match.” Incoming funds will not be in an expected-id set; if GCM fails there is no payload.

The extra SHA-256 checksum header is a **weaker** signal than GCM. Official samples disagree on header names and key encoding.

Mutual IP allowlists shrink the network attacker set. They do **not** replace GCM. Azure source IP lists change.

The first crypto or delivery failure is a **same-day page** (daily settlement). Do not wait for the ~3h40m warning email or auto-deactivate.

Webhooks are a hint. Drift control is the reconciliation report.

See [Setup webhooks](setup-webhooks.md) and [Troubleshooting](troubleshooting.md). Retry behaviour: [Webhook retry strategy](https://docs.bankingcircleconnect.com/docs/webhook-retry-strategy). Reconciliation: [Reconciliation using webhooks](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks).

Payouts that originate in B4B Oversight: book only after Banking Circle `Processed` or a recon line. Oversight `B4BTMApproved` is handoff, not settlement. [Oversight payment tracking](../../b4b-payments/oversight-payment-tracking.md).

## Does B4B Oversight mean the payment is settled at Banking Circle?

No. Oversight is regulatory / pre-settlement only. Terminal Oversight statuses are `B4BTMApproved` (handed to BC, BC accepted) or `B4BFailed`. Settlement updates come from Banking Circle. At handoff, store `banking_circle_api_response.paymentId`. BC has no `Settled` status (`Processed` = sent to recipient; `Booked` = account event). Full note: [Oversight payment tracking](../../b4b-payments/oversight-payment-tracking.md).
