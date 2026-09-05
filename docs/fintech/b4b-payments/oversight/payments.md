# Oversight payments

Sources: `beneficiaries-and-payments.md`, `beneficiaries-and-payments-KEY-EXTRACTS.md`, `createpayment-1.md` (Oversight OpenAPI).

> **Warning — do not conflate with on-platform create payment.**  
> Oversight create is documented in local `createpayment-1.md` (`POST https://…/oversight/v1/payments`, Bearer JWT).  
> On-platform `createpayment.md` is a **different** API (`/services/json`, Basic Auth, title “Payments”, staging/production hosts without `/oversight`). Do not mix request shapes, statuses, or auth.

## Model

Every payment is from a **Company** to a **Beneficiary**. Both live in Oversight (`company_id`, `beneficiary_id`). The debit account is a **vIBAN** registered against the company. The payments endpoint is a **passthrough plus extras**: Banking Circle singles fields (`amount`, `currencyOfTransfer`, `debtorViban`, `creditorAccount`, etc.) are forwarded to BC; B4B wrapper fields sit at the top level and are extracted first.

Base URL used in guide examples: `https://sandbox.b4bpayments.com/oversight/v1`.

## Create payment

`POST /payments`

### B4B wrapper fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `external_ref` | string | yes | Unique within client scope |
| `beneficiary_id` | UUID | yes | Beneficiary with consolidated `sanctions_status` currently `pass` |
| `company_id` | UUID | yes | Must own the vIBAN in `debtorViban.account` |
| `callback_url` | URL | yes | HTTPS; regulatory-phase status updates only |
| `sca_applied` | boolean | conditional | Required when client flag `require_sca_applied` is enabled |
| `sca_exemption_reason` | enum | conditional | Required if `sca_applied` is `false` |

SCA exemption enum (from OpenAPI / guide): `trusted_beneficiaries`, `recurring_payment`, `contactless_low_value`, `unattended_terminals`, `low_value`, `secure_corporate_payment`, `transaction_risk_analysis`, `merchant_initiated_transaction`, `other`.

### Common BC passthrough fields

| Field | Notes |
|---|---|
| `amount.amount` / `amount.currency` | Numeric amount; ISO 4217 |
| `currencyOfTransfer` | Delivery currency to creditor |
| `debtorViban.account` | vIBAN belonging to `company_id` |
| `creditorAccount.account` | Must equal beneficiary `account_number` |
| `creditorAccount.financialInstitution` | Must equal beneficiary `financial_institution` (incl. `SC` / `FW` prefix) |
| `creditorName` | Must equal beneficiary `account_name` |
| `remittanceInformation.line1` | Used in TM as payment reference |
| `chargeBearer` | `SHA`, `BEN`, or `OUR` (default `SHA`) |
| `requestedExecutionDate` | ISO 8601 datetime |

Creditor fields are a **consistency check**, not a recipient override. Mismatch → `422`, nothing forwarded to BC.

### Example request (GBP Faster Payments; placeholders)

```json
{
  "external_ref": "{YOUR_PAYMENT_REF}",
  "beneficiary_id": "{beneficiary_id}",
  "company_id": "{company_id}",
  "callback_url": "https://example.com/webhooks/b4b/payments",
  "sca_applied": true,
  "requestedExecutionDate": "2026-05-27T00:00:00+00:00",
  "amount": { "currency": "GBP", "amount": 1500.00 },
  "currencyOfTransfer": "GBP",
  "chargeBearer": "SHA",
  "debtorViban": { "account": "{DEBTOR_VIBAN}" },
  "creditorAccount": {
    "account": "{CREDITOR_ACCOUNT}",
    "financialInstitution": "SC112233"
  },
  "creditorName": "{CREDITOR_LEGAL_NAME}",
  "remittanceInformation": { "line1": "Invoice {INVOICE_REF}" }
}
```

### Successful response (`201 Created`)

```json
{
  "id": "{payment_id}",
  "status": "B4BAccepted",
  "payload": {
    "amount": { "amount": 1500.00, "currency": "GBP" },
    "currencyOfTransfer": "GBP",
    "debtorViban": { "account": "{DEBTOR_VIBAN}" },
    "creditorAccount": {
      "account": "{CREDITOR_ACCOUNT}",
      "financialInstitution": "SC112233"
    },
    "creditorName": "{CREDITOR_LEGAL_NAME}",
    "debtorReference": "{DEBTOR_REFERENCE}"
  }
}
```

`payload` echoes the BC-shaped body after validation. OpenAPI also documents `401` (missing/invalid token) and `422` (validation errors).

### Sample `422` bodies

```json
{
  "errors": {
    "beneficiary": ["Beneficiary failed sanctions check"]
  }
}
```

```json
{
  "errors": {
    "base": [
      "Beneficiary account name does not match with BC payload"
    ]
  }
}
```

## Statuses (six)

Oversight publishes only these on its payment webhook:

| Status | Meaning |
|---|---|
| `B4BAccepted` | Accepted and saved; pre-screening |
| `B4BSanctionsPending` | Payment-level sanctions possible match; under review |
| `B4BSanctionsApproved` | Sanctions cleared; moving to TM |
| `B4BTMPending` | Transaction monitoring flagged for review |
| `B4BTMApproved` | TM cleared; **handed off to Banking Circle**. Terminal from Oversight |
| `B4BFailed` | Will not send; sanctions/TM rejection or BC refused at handoff. Terminal |

Typical happy path:

```
B4BAccepted → B4BSanctionsApproved → B4BTMApproved → [handoff to BC]
```

Other documented paths include sanctions/TM pending→approved or →`B4BFailed`, and BC refusal at handoff ending in `B4BFailed` with BC error in `banking_circle_api_response`.

### Handoff and `banking_circle_api_response.paymentId`

`B4BTMApproved` means the payment was submitted to BC and BC accepted it for processing. From that moment:

- `banking_circle_api_response.paymentId` is the BC payment id for BC webhooks, reports, MT103, recalls/traces.
- Settlement tracking is BC’s responsibility.

If BC’s `/api/v1/payments/singles` call fails at handoff, status stays terminal `B4BFailed` with the BC error in `banking_circle_api_response`; the payment never enters BC’s pipeline and no BC webhook fires for it.

## `callback_url` behaviour

On each regulatory-phase status change, Oversight POSTs to the `callback_url` from the create request. Final Oversight status is either `B4BTMApproved` or `B4BFailed`.

**Webhook stops at handoff.** Settlement states (processed, returned, rejected by receiving bank, recall) come from **Banking Circle** webhooks, correlated on the surfaced BC `paymentId`.

Example callback body at handoff (placeholders):

```json
{
  "id": "{payment_id}",
  "status": "B4BTMApproved",
  "payload": {
    "amount": { "amount": 1500.00, "currency": "GBP" },
    "currencyOfTransfer": "GBP",
    "debtorViban": { "account": "{DEBTOR_VIBAN}" },
    "debtorAccount": null,
    "creditorAccount": {
      "account": "{CREDITOR_ACCOUNT}",
      "financialInstitution": "SC112233"
    },
    "creditorName": "{CREDITOR_LEGAL_NAME}",
    "debtorReference": "{DEBTOR_REFERENCE}"
  },
  "banking_circle_api_response": {
    "paymentId": "{banking_circle_payment_id}",
    "status": "PendingProcessing"
  }
}
```

Delivery (summary): HTTPS POST JSON; Oversight payment callbacks **not currently signed**; polynomial backoff on 5xx/network error, stop after roughly a day; no manual replay; may deliver same status twice; ordering not guaranteed; any `2xx` acknowledges. Full detail: [webhooks-and-callbacks.md](webhooks-and-callbacks.md).

## Lookups

### `GET /payments/{id}`

Fetches a single payment, including `banking_circle_api_response` when available. Use when webhooks are unreliable or for request/response flows.

### `GET /payments`

Paginated list. Useful query parameters:

| Parameter | Notes |
|---|---|
| `page` | Default 1 |
| `per_page` | Default 50 |
| `status` | One of the six Oversight statuses |
| `external_ref` | Filter by integrator reference (direct lookup) |

These return **Oversight** statuses only (terminal success = `B4BTMApproved`), plus `banking_circle_api_response.status` as at handoff (e.g. `PendingProcessing`). They do **not** return settled status.

## Beneficiaries (prerequisite)

Payments require an approved beneficiary (`POST /beneficiaries`, `GET /beneficiaries/{id}`, sanctions via beneficiary `callback_url`). Consolidated `sanctions_status` must be `pass` at payment create time. See source guide Part 1; beneficiary webhook detail is under [webhooks-and-callbacks.md](webhooks-and-callbacks.md).

## OpenAPI note (`createpayment-1.md`)

OpenAPI title: **Oversight API** v2.0.0. Path: `POST /payments`. Servers: sandbox and production `/oversight/v1`. Callbacks: `paymentStatusChange` → `{$request.body#/callback_url}` with `PaymentCallback` schema (Payment + optional `banking_circle_api_response`).
