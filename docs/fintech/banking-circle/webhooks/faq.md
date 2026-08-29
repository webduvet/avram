# FAQ

## If Banking Circle fails to deliver a webhook, is the payment reversed?

No. Webhooks are notifications. Payment status `Reversed` means the **scheme rejected the payment**, not that a webhook failed. Failed delivery retries, then deactivates the subscription; it does **not** unwind the booking. Reconciliation reports and `GET` payments are the source of truth.

Sources: [Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments) · [Webhook retry strategy](https://docs.bankingcircleconnect.com/docs/webhook-retry-strategy) · [Reconciliation using webhooks](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks)
