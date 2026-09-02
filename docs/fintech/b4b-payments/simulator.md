# Oversight payment simulator

Contract for a **fake B4B Oversight**, regulatory / pre-settlement only. Public developer portal is sales-gated. **Do not invent OpenAPI paths** that are not in the Avram notes. Mark every route **simulated** vs **still vendor-shaped**.

Sources: [Oversight payment tracking](oversight-payment-tracking.md) · [Integrator baseline](integrator-baseline.md) · [Questions and undocumented features](questions-and-undocumented-features.md) · [Beneficiaries and Payments](https://b4bpayments.readme.io/docs/beneficiaries-and-payments)

## Must implement

| Behaviour | Simulator rule |
|---|---|
| Phase | Oversight = **pre-settlement only**. Six statuses. Terminals: `B4BTMApproved` (handed to BC **and** BC accepted), `B4BFailed`. |
| Webhook | Oversight webhook **stops at handoff**. Settlement updates are **Banking Circle**, not B4B. |
| Ids at handoff | On `B4BTMApproved`, persist `external_ref` → B4B id → `banking_circle_api_response.paymentId`. That BC id is what the [connected path](../simulation/connected-path.md) uses. |
| Connect subscription | **Integrator-owned.** Payment-create does **not** auto-subscribe (documented). Do **not** build the sim as if Oversight registers `…/banking_circle/push_notifications`. Observed staging URL is undocumented. |
| Extra callback | A later Oversight POST to `callback_url` when BC is `Processed` has been **seen in the wild**, unsigned, undocumented. Simulator **may** emit it behind a flag, **default OFF**, labelled **opportunistic**. Not the settlement contract. |
| Rails (v1) | FPS / SEPA / SWIFT / ACH as **status fields only**. No scheme simulation. |

## APIs to stub (paths from notes only)

| Surface | Notes |
|---|---|
| `POST` payment | Simulated. Idempotency **only if** already in our notes; do not invent a key header. |
| `GET /payments/{id}` | Simulated. Regulatory-phase only — **not** settled on a safeguarding account. |
| `GET /payments?external_ref=` | Simulated, paginated list. Same: not settlement. |
| Auth | If the lab needs B4B-shaped auth: **OAuth 2.0 customer-credentials**, token **60 minutes**, refresh. Public marketing only — **vendor-shaped**, no published token URL. |

No other paths. No on-platform `/services/json/payments` mixed into Oversight (that API **does** document `Processed` on `callback_url` and is a different product).

## Simulated vs vendor-shaped

| Simulated (must behave) | Still vendor-shaped (no public schema — stub thinly) |
|---|---|
| Status machine through `B4BTMApproved` / `B4BFailed` | Full OpenAPI, error catalogue, KYB pack |
| Handoff id mapping | Exact `banking_circle_api_response` JSON |
| Webhook stops at handoff | Wire header names, signatures |
| Optional extra callback, default off | Treating that callback as product |

## Lab success

`POST` a payment with `external_ref` → statuses reach `B4BTMApproved` → BC-shaped `paymentId` exists → Oversight webhook has **stopped**. Settlement is the [BC webhook simulator](../banking-circle/webhooks/simulator.md), not this service.
