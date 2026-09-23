# Reverse vs Return (Banking Circle Connect)

How Banking Circle Connect models **scheme reverse** vs **beneficiary return** for outgoing payouts.  
Related: [Payment lifecycle](payment-lifecycle.md) · [Payment confirmation](payment-confirmation.md) · [Midday recon sweep](midday-recon-sweep.md)

**Labels:** **Documented** = Connect primary docs. **Observed** = integrator/buddy behaviour (not in Connect docs).

## One-line distinction

| | Reverse | Return |
| --- | --- | --- |
| Who sends money back | Payment **scheme** (after BC had marked the payout Processed) | **Recipient’s bank** (beneficiary return) |
| Original payout status | Becomes **`Reversed`** (same `paymentId`) | Stays **`Processed`** |
| Money-back shape | Second booking on the **same** payment (negative debit effect) | **New incoming** payment (`return: true`) |

“Returned” is **not** a Connect payment status ([Payment status](https://docs.bankingcircleconnect.com/docs/payment-status)).

## Reverse (Documented)

**What:** Rare case: payout was `Processed`, then rejected by the payment scheme → status `Reversed` (final).  
Sources: [Payment status](https://docs.bankingcircleconnect.com/docs/payment-status), [Outgoing payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments).

**Money back:** Two `OutgoingPaymentBooked` notifications — initial book, then reverse. Same account and transaction references; amount reflects the balance effect. Recon: for a debit reversal, `debitAmount` is the **negative** of the original debit ([Payload examples](https://docs.bankingcircleconnect.com/docs/payload-examples) OutgoingPaymentBooked callout; [Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report)).

**Webhooks:** `OutgoingPaymentBooked` (×2) + `Reversed` on the **same** `paymentId`. Scenario walkthrough: [Practical guide — Scenario 5](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks).

**Reports:** Expect reverse booking evidence on **reconciliation** (negative DBIT; optional `statusReasonCode` / `statusReasonDescription`). Use **GET payment/status** and webhooks. **Rejection Report** defaults are Missing funding / Pending processing / Rejected only — not scheme `Reversed`. OpenAPI `IncludeReversals` is **Direct Debit / SEPA DD only**, not CT scheme reverse.

**Sandbox:** amount `27.00` → Reversed after ~30s ([Simulating payments](https://docs.bankingcircleconnect.com/docs/simulating-payments)).

**Observed (buddy):** marks payout REVERSED and posts a ledger reversal. *(Integrator note — verify in buddy handlers.)*

## Return (Documented)

**What:** Recipient’s financial institution returns the funds (e.g. closed/invalid account). Original outgoing remains `Processed`.  
Sources: [Outgoing payments — Returned Payments](https://docs.bankingcircleconnect.com/docs/outgoing-payments), [Incoming payments](https://docs.bankingcircleconnect.com/docs/incoming-payments), [Payment status](https://docs.bankingcircleconnect.com/docs/payment-status).

**Money back:** Treated as **two payments** — original outgoing + **new incoming**. Incoming carries `"return": true` on `IncomingPaymentProcessed` / GET payments / recon. Remittance often `RETURN OF PAYMENT` plus original outgoing **transaction reference** ([Practical guide — Returns](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks); [Payload examples — Returned payments](https://docs.bankingcircleconnect.com/docs/payload-examples)).

**Webhooks:** `IncomingPaymentProcessed` (`return: true`) and `IncomingPaymentBooked` for the **new** payment. No dedicated Returned event. Do **not** rely on Booked→Processed order (happy-path scenarios show Processed then Booked; order not guaranteed).

**Recon hints (optional extras):** `return: true`; remittance may include `/RETN/`, reason e.g. `/AC04/Closed Account Number`, original ref `/MREF/…`; `bankTrnsCodeSubFamily` may be `RRTN` (“Reversal due to Payment Return”) — naming overlaps “reverse”; trust status + `return` flag first ([Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report)).

**Midday outbound confirm:** match **`DBIT`** rows with **`return` null**; `return: true` is an incoming return, not confirmation of the original outbound ([Midday recon sweep](midday-recon-sweep.md)).

**Sandbox:** amount `177.00` → returned (with fee scenario) ([Simulating payments](https://docs.bankingcircleconnect.com/docs/simulating-payments)).

**Observed (buddy):** may leave original payout SUCCESS while funds sit back at BC as a separate incoming. *(Integrator note — verify in buddy handlers. Consistent with Documented original=`Processed`.)*

## Integrator checklist

1. Subscribe to `Reversed`, `OutgoingPaymentBooked`, `IncomingPaymentProcessed`, `IncomingPaymentBooked` (plus processed/rejected as needed).
2. On **`Reversed`**: unwind payout success / ledger — same `paymentId`.
3. On incoming `return: true`: do **not** expect original status change; link via remittance / original transaction reference; decide product policy (Observed: buddy may currently ignore).
4. Recon sweep: outbound success ≈ `DBIT` + `return` null + processed evidence; never treat `return: true` as outbound confirm.
5. Do not use Rejection Report as the place to find scheme `Reversed` CTs.

## Primary Connect sources

- https://docs.bankingcircleconnect.com/docs/payment-status
- https://docs.bankingcircleconnect.com/docs/outgoing-payments
- https://docs.bankingcircleconnect.com/docs/incoming-payments
- https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks
- https://docs.bankingcircleconnect.com/docs/payload-examples
- https://docs.bankingcircleconnect.com/docs/reconciliation-report
- https://docs.bankingcircleconnect.com/docs/webhooks
- https://docs.bankingcircleconnect.com/docs/simulating-payments

## Verification caveats

- Timing phrases such as returns arriving “usually days later” are **Undocumented** in Connect primary pages used for this note.
- Connect path `/docs/returns` was password-gated at verification time (2026-09-23); claims above rely on the listed primary sources, not that page.
- Buddy behaviour rows are **Observed** only — verify against live integrator handlers before treating as product truth.
