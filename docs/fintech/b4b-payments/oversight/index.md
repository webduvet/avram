# B4B Payments — Oversight API

Regulatory pre-settlement API used internally in front of Banking Circle (BC). Oversight accepts payments, runs sanctions screening and transaction monitoring, then either rejects the payment or hands it off to BC for settlement. Settlement status after handoff is BC’s responsibility.

**Base URLs (from source OpenAPI):**

| Environment | Base URL |
|---|---|
| Sandbox | `https://sandbox.b4bpayments.com/oversight/v1` |
| Production | `https://b4bpayments.com/oversight/v1` |

**Upstream docs:** [https://b4bpayments.readme.io](https://b4bpayments.readme.io) (password-gated). Derived from a local mirror of the password-gated B4B docs.

## Pages in this extract

| Page | Contents |
|---|---|
| [Authentication](authentication.md) | Oversight JWT (RS512) auth; contrast with on-platform Basic Auth |
| [Payments](payments.md) | Create payment, six statuses, lookups, handoff `paymentId` |
| [Webhooks and callbacks](webhooks-and-callbacks.md) | Payment/beneficiary `callback_url` behaviour; signing notes |
| [Documents and onboarding](documents-and-onboarding.md) | Company setup, document upload, extended profile, vIBANs |
| [Reports (adjacent)](reports.md) | Post-handoff BC reports / on-platform extracts — not Oversight APIs |

## Related conceptual notes (elsewhere in this tree)

These pages are expected to exist as sibling notes under `docs/fintech/b4b-payments/` (not rewritten here):

- [`../oversight-payment-tracking.md`](../oversight-payment-tracking.md) — payment tracking across Oversight → BC
- [`../questions-and-undocumented-features.md`](../questions-and-undocumented-features.md) — open questions and undocumented behaviour (including any post-handoff callback beyond documented Oversight webhooks)

## Do not conflate APIs

| Surface | Path / auth | Role |
|---|---|---|
| **Oversight** | `/oversight/v1`, Bearer JWT (RS512) | Regulatory pre-settlement, company/beneficiary KYC intake |
| **On-platform web services** | `/services/json/...`, HTTP Basic Auth | Card/float/payments on the B4B platform (separate OpenAPI: `createpayment.md`, not Oversight `createpayment-1.md`) |
| **Banking Circle Connect** | BC mTLS APIs + BC webhooks | Settlement, returns, recalls, rails after handoff |

## Scope reminder

Oversight webhook coverage ends at handoff (`B4BTMApproved` or `B4BFailed`). For settled / returned / rejected-by-receiver status, integrate with Banking Circle’s notification stream and correlate on `banking_circle_api_response.paymentId`.
