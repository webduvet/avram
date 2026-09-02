# Questions and undocumented features

Standing list of integrator observations that **contradict or go beyond** published B4B Oversight and Banking Circle Connect docs. Each item is labelled **Documented** / **Observed** / **Open**. Lifecycle and booking rules stay on the existing pages — this is the questions bucket.

Related: [Oversight payment tracking](oversight-payment-tracking.md) · [Payment confirmation](../banking-circle/payment-confirmation.md) · [Webhooks FAQ](../banking-circle/webhooks/faq.md)

## 1. Extra Oversight callback after handoff

| | |
|---|---|
| **Documented** | Oversight webhook covers the **regulatory phase only**. Terminal statuses are `B4BTMApproved` or `B4BFailed`. Settlement updates come from **BC webhooks**, not Oversight. The integrator must subscribe to BC. |
| **Observed** | The same client `callback_url` also fires when BC **processes** the payment (not only at regulatory/handoff). |
| **Open** | Will B4B productize this? Until the published page changes, **do not treat it as the settlement contract**. |

Sources: [Beneficiaries and Payments](https://b4bpayments.readme.io/docs/beneficiaries-and-payments) (lifecycle, `callback_url` field, [payment webhook callbacks](https://b4bpayments.readme.io/docs/beneficiaries-and-payments#payment-webhook-callbacks), [webhook stops at the handoff](https://b4bpayments.readme.io/docs/beneficiaries-and-payments#our-webhook-stops-at-the-handoff), after-handoff step 1). OpenAPI [create payment](https://b4bpayments.readme.io/reference/createpayment-1): callback is `POST` to `{$request.body#/callback_url}`; `InternalPaymentStatus` is `B4B*` only; `banking_circle_api_response` is untyped; the guide example status is `PendingProcessing`.

## 2. Connect subscription pointing at a B4B staging path

| | |
|---|---|
| **Observed** | A live Connect subscription whose endpoint is `https://staging.b4bpayments.com/banking_circle/push_notifications` (not an integrator-configured URL). Shape matches a normal Connect subscription: status `2` Active, `maxNotificationsPerMessage` `1000` (portal default), `mtlsEnabled` false, events `IncomingPaymentProcessed`, `OutgoingPaymentRejected`, `OutgoingPaymentProcessed`, `MissingFunding`, `Reversed`, `targetType` `1` = Company. |
| **Documented** | That path does **not** appear in B4B ReadMe or Connect docs (zero hits). Oversight does **not** say it creates, updates, or deletes Notification Self Service subscriptions. Connect: the caller defines the URL; unlimited subscriptions; unique URLs only; `POST` of a different URL does **not** delete another. URL change/removal: `PUT` endpoint, `DELETE` + `If-Match`, or portal Remove. `targetType`: `0` Account / `1` Company / `2` CompanyGroup. |
| **Open** | Who holds the Connect API user / portal that can `DELETE` or `PUT` this subscription? Does Oversight silently register this URL (unpublished)? |

Sources: [POST subscription](https://docs.bankingcircleconnect.com/reference/post_api-v1-notificationselfservice-subscription) · [PUT subscription](https://docs.bankingcircleconnect.com/reference/put_api-v1-notificationselfservice-subscription-id) · [DELETE subscription](https://docs.bankingcircleconnect.com/reference/delete_api-v1-notificationselfservice-subscription-id) · [Webhook subscriptions](https://docs.bankingcircleconnect.com/docs/webhook-subscriptions)

Do not conflate with the next item.

## 3. Ambiguous published sentence

| | |
|---|---|
| **Documented (ambiguous)** | “[The rail that was actually used appears on the payment callback once the payment is processed](https://b4bpayments.readme.io/docs/beneficiaries-and-payments#routing-identifier-formats).” Does **not** name Oversight vs BC callback. The same page later says `paymentRail` comes from **BC’s webhook**. |
| **Open** | Which callback is “the payment callback”? Until clarified, treat rail as a **BC webhook** field, not an Oversight settlement signal. |

## 4. Do not conflate the on-platform payments API

| | |
|---|---|
| **Documented** | [Create payment (on-platform)](https://b4bpayments.readme.io/reference/createpayment) uses host `staging.b4bpayments.com` `/services/json/payments` and **does** document `Processed` on `callback_url`. That is **not** Oversight `POST /payments`. |

Mixing the two APIs is how “B4B already sends Processed” gets misread onto Oversight.

## How to use this list

Booking and sweep rules are unchanged: [Payment confirmation](../banking-circle/payment-confirmation.md). Oversight handoff vs Connect auto-subscribe: [Oversight payment tracking](oversight-payment-tracking.md#does-oversight-create-a-connect-webhook-subscription). FAQ: [Does Oversight auto-subscribe a Connect webhook?](../banking-circle/webhooks/faq.md#does-oversight-auto-subscribe-a-connect-webhook).
