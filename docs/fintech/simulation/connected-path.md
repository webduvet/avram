# Connected path

The point of the lab: a fake Oversight payment that reaches `B4BTMApproved` **must** create a BC-shaped `paymentId` **and** **must** cause the webhook simulator to deliver `OutgoingPaymentProcessed` / `Booked` (or a **failed** delivery that **retries**). Isolated `clienttest` / ping is **not** enough.

Implementers: [B4B simulator](../b4b-payments/simulator.md) · [BC webhook simulator](../banking-circle/webhooks/simulator.md). Booking rules: [Payment confirmation](../banking-circle/payment-confirmation.md) · [Oversight tracking](../b4b-payments/oversight-payment-tracking.md).

## Flow

1. Create payment on the **B4B sim** (`external_ref`).
2. Statuses run until `B4BTMApproved` (or `B4BFailed` — stop; no BC payment).
3. At handoff a **BC sim payment** exists with that `paymentId`.
4. Notifier **POSTs** to the receiver: prefer `new_event` `OutgoingPaymentProcessed` and, separately / unordered, `Booked`. If the receiver fails, **`retry_failed`** — not a silent drop.
5. Optional ultimate confirm: BC `GET` status / recon. BC has **no `Settled`**. `Processed` = sent to recipient. `Booked` = account event. Order not guaranteed.

```
B4B sim  --handoff-->  BC sim paymentId  --notifier-->  receiver
     |                         |
     webhook stops             GET status / recon (source of truth)
```

## What must not happen

- Booking the **customer / virtual ledger** on Oversight approval alone.
- Treating `replay_mock` / `clienttest` as proof the connected path works.
- Oversight auto-creating a Connect subscription.
- Extra Oversight “Processed” callback as the settlement signal (flag off unless a lab is testing that observation).

## Lab success

One `external_ref` can be traced: B4B id → BC `paymentId` → decryptable webhook POST (or documented retries) → sweep/GET can see `Processed` or `Rejected`. The settlement worker is the only ledger writer; webhook ACK ≠ book.
