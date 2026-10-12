# Reversed vs rejected payment (Banking Circle Connect)

How Banking Circle Connect distinguishes a **rejected** outgoing payment from a **reversed** one, with **returned** as a contrast.  
Related: [Reverse vs Return](reverse-vs-return.md) · [Payment lifecycle](payment-lifecycle.md) · [Midday recon sweep](midday-recon-sweep.md)

**Labels:** **Documented** = Connect primary docs (verified against the local Connect mirror). **Interpretation** = reading of the docs, not stated literally. **Observed / integrator** = integrator guidance, not in Connect docs.

## Summary

| | Rejected | Reversed | Returned (contrast) |
| --- | --- | --- | --- |
| Who rejects / sends back | BC (validation, funding, etc.) | Payment **scheme**, after BC marked it `Processed` | **Beneficiary bank** |
| Original status | `Rejected` (final) | `Processed` → `Reversed` (final) | Stays `Processed` |
| `paymentId` | Original | Same original | **New** incoming payment |
| Rejection Report | Yes | No | No |
| Intraday recon | No | Yes (booking + reverse booking) | Yes, as incoming with `return: true` |
| Sandbox amount | `19.00` | `27.00` (after 30 s) | `177.00` → returns `147.00` (after 30 s) |

## Rejected (Documented)

- Terminal: `Rejected` is a final state; the payment never processes ([Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments), [Payment status](https://docs.bankingcircleconnect.com/docs/payment-status)).
- Visible via **GET payment/status** and the `OutgoingPaymentRejected` webhook ([Webhooks](https://docs.bankingcircleconnect.com/docs/webhooks)).
- Shows on the **Rejection Report**, which by default covers Missing funding / Pending processing / Rejected.
- **Not in intraday recon**: reconciliation reports (including intraday) include only payments with status `processed` and `pendingProcessing` ([Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report)). Consistent with [Midday recon sweep](midday-recon-sweep.md).
- Sandbox: amount `19.00` → rejected immediately ([Simulating payments](https://docs.bankingcircleconnect.com/docs/simulating-payments)).

## Reversed (Documented)

- The payment scheme rejects the payment after BC has already marked it `Processed`; BC describes this as happening "in rare cases" ([Payment status](https://docs.bankingcircleconnect.com/docs/payment-status)). `Reversed` is a final state ([Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments)).
- Same original `paymentId`.
- Sequence: `OutgoingPaymentBooked` + `OutgoingPaymentProcessed` (order not guaranteed; Scenario 5 shows Booked before Processed), then a **second** `OutgoingPaymentBooked` for the reverse booking (same account and transaction references; amount reflects the balance effect, i.e. negative debit), then status `Reversed` with the **`Reversed`** webhook ([Practical guide — Scenario 5](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks), [Payload examples](https://docs.bankingcircleconnect.com/docs/payload-examples), [Webhooks](https://docs.bankingcircleconnect.com/docs/webhooks)).
- The webhook event name is **`Reversed`** (subscription event type 5). There is **no** `OutgoingPaymentReversed` event in the Connect docs or OpenAPI.
- Not on the Rejection Report. Seen via webhooks, GET status (optional `statusReasonCode` / `statusReasonDescription`), and reconciliation (reverse booking) ([Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report)).
- Sandbox: amount `27.00` → reversed after 30 seconds ([Simulating payments](https://docs.bankingcircleconnect.com/docs/simulating-payments)).
- **Interpretation:** in practice the funds never reach (or are not kept by) the beneficiary. The docs say "rejected by the payment scheme", not literally "never credited".

## Returned — contrast (Documented)

- The original outgoing stays `Processed`. The beneficiary bank sends the funds back as a **new incoming payment** with its own `paymentId` and `return: true` ([Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments), [Incoming payments](https://docs.bankingcircleconnect.com/docs/incoming-payments)).
- No documented field holds the original `paymentId`. The link is the remittance text: "in most cases" `RETURN OF PAYMENT` plus a reference to the original outgoing payment and a reason, with optional `/RETN/` or `bankTrnsCodeSubFamily` `RRTN` hints ([Payload examples](https://docs.bankingcircleconnect.com/docs/payload-examples), [Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report)).
- Matching therefore means parsing remittance against your own reference; the format is not guaranteed to be structured.
- Sandbox: amount `177.00` → returned with fee deduction, `147.00` comes back after 30 seconds ([Simulating payments](https://docs.bankingcircleconnect.com/docs/simulating-payments)).
- Details: [Reverse vs Return](reverse-vs-return.md).

## Finality

- **Documented:** final states are `Rejected`, `Reversed` and `Cancelled` ([Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments)). The docs also call `Processed` "the final state", yet a `Processed` payment can later become `Reversed` ([Payment status](https://docs.bankingcircleconnect.com/docs/payment-status)).
- **Documented (by absence):** the docs give no time window after which `Processed` can no longer be reversed.
- **Interpretation:** `Processed` means done unless a reversal arrives.
- **Observed / integrator:** book on `Processed` and keep an undo handler for the `Reversed` webhook on the same `paymentId`.
- **Open question for BC:** is there a hard cutoff after which `Processed` is final?

## Sources

- https://docs.bankingcircleconnect.com/docs/payment-status
- https://docs.bankingcircleconnect.com/docs/outgoing-payments
- https://docs.bankingcircleconnect.com/docs/incoming-payments
- https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks
- https://docs.bankingcircleconnect.com/docs/payload-examples
- https://docs.bankingcircleconnect.com/docs/reconciliation-report
- https://docs.bankingcircleconnect.com/docs/webhooks
- https://docs.bankingcircleconnect.com/docs/simulating-payments
