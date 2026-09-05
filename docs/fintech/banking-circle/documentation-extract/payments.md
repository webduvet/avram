# Payments (status and related GETs)

Single-payment GET endpoints from the local Connect OpenAPI mirror, plus payment status meanings from the guides.

**Reference OpenAPI:** local Connect reference mirror. Live base URL: `https://www.bankingcircleconnect.com` (sandbox: `https://sandbox.bankingcircleconnect.com`). Auth: Bearer token.

Related write endpoints mentioned in guides (not fully extracted here):

- `POST /api/v1/payments/singles` (initiate)
- `PUT /api/v1/payments/singles/{payment-id}/cancel`
- `PUT /api/v1/payments/singles/{payment-id}/reject` (direct debit / 3rd-party)

---

## GET `/api/v1/payments/singles/{payment-id}`

Get details of a payment by payment id (includes status and full payment payload).

- **Source:** `reference/get_api-v1-payments-singles-payment-id.md` — `updatedAt: 2026-06-08T06:01:06.000Z`
- **URL:** https://docs.bankingcircleconnect.com/reference/get_api-v1-payments-singles-payment-id
- **Tags:** Single payments

### Parameters

| Name | In | Required | Type | Description |
| --- | --- | --- | --- | --- |
| `payment-id` | path | yes | string (uuid) | Payment identifier (`Payment.paymentId`) |

### Request example

```http
GET https://www.bankingcircleconnect.com/api/v1/payments/singles/{payment-id}
Authorization: Bearer {access-token}
Accept: application/json
```

```bash
curl --request GET \
  --url 'https://www.bankingcircleconnect.com/api/v1/payments/singles/{payment-id}' \
  --header 'Accept: application/json' \
  --header 'Authorization: Bearer {access-token}'
```

### Response example (`200` — Payment)

IDs and account numbers redacted to placeholders from OpenAPI sample.

```json
{
  "paymentId": "{payment-id}",
  "transactionReference": "{transaction-reference}",
  "concurrencyToken": "{concurrency-token}",
  "classification": "Outgoing",
  "subClassification": "Internal",
  "isTraceable": true,
  "status": "Processed",
  "processedTimestamp": "2024-06-14T11:17:42+00:00",
  "latestStatusChangedTimestamp": "2024-06-14T11:17:42+00:00",
  "booked": true,
  "reversalBooked": null,
  "return": null,
  "paymentRail": "Internal",
  "errors": [],
  "lastChangedTimestamp": "2024-06-14T11:17:42+00:00",
  "debtorInformation": {
    "paymentBulkId": null,
    "accountId": "{account-id}",
    "account": {
      "accountNumber": "{account-number}",
      "accountIban": "{account-iban}",
      "account": "{account-iban}",
      "financialInstitution": "{bic}",
      "country": null
    },
    "vibanId": null,
    "viban": null,
    "instructedDate": "2024-06-14T00:00:00+00:00",
    "debitAmount": {
      "currency": "DKK",
      "amount": 5.75
    },
    "debitValueDate": "2024-06-14T00:00:00+00:00",
    "fxRate": null,
    "instruction": {
      "debtorAccount": {
        "accountNumber": "{account-number}",
        "accountIban": "{account-iban}",
        "account": "{account-iban}",
        "financialInstitution": null,
        "country": null
      },
      "debtorViban": null,
      "debtorReference": "TEST REFERENCE 1",
      "debtorNarrativeToSelf": null,
      "currencyOfTransfer": "DKK",
      "amount": {
        "currency": "DKK",
        "amount": 5.75
      },
      "requestedExecutionDate": "2024-06-14T00:00:00+00:00",
      "chargeBearer": "SHA",
      "remittanceInformation": {
        "line1": "Remittance information line 1",
        "line2": "Remittance information line 2",
        "line3": "Remittance information line 3",
        "line4": "Remittance information line 4"
      },
      "creditorId": null,
      "creditorAccount": {
        "accountNumber": "{creditor-account-number}",
        "accountIban": "{creditor-iban}",
        "account": "{creditor-iban}",
        "financialInstitution": "{bic}",
        "country": "DK"
      },
      "creditorName": "CREDITOR NAME",
      "creditorAddress": {
        "line1": "CREDITOR ADDRESS 1",
        "line2": "CREDITOR ADDRESS 2",
        "line3": null
      },
      "instructedChargeBearer": "SHA",
      "clearingNetwork": null,
      "debtorName": null,
      "debtorAddress": null,
      "ultimateDebtorAccount": null,
      "ultimateDebtorName": null,
      "ultimateDebtorAddress": null,
      "clientCustomerId": null
    }
  },
  "transfer": {
    "debtorAccount": {
      "accountNumber": "{account-number}",
      "accountIban": "{account-iban}",
      "account": "{account-iban}",
      "financialInstitution": "{bic}",
      "country": "DK"
    },
    "debtorName": "DEBTOR NAME",
    "debtorAddress": null,
    "amount": {
      "currency": "DKK",
      "amount": 5.75
    },
    "valueDate": "2024-06-14T00:00:00+00:00",
    "chargeBearer": "SHA",
    "remittanceInformation": {
      "line1": "Remittance information line 1",
      "line2": "Remittance information line 2",
      "line3": "Remittance information line 3",
      "line4": "Remittance information line 4"
    },
    "additionalRemittanceInformation": null,
    "creditorAccount": {
      "accountNumber": "{creditor-account-number}",
      "accountIban": "{creditor-iban}",
      "account": "{creditor-iban}",
      "financialInstitution": "{bic}",
      "country": "DK"
    },
    "creditorName": "CREDITOR NAME",
    "creditorAddress": {
      "line1": "CREDITOR ADDRESS 1",
      "line2": "CREDITOR ADDRESS 2",
      "line3": null
    },
    "ultimateCreditorAccount": null,
    "instructedChargeBearer": "SHA"
  },
  "creditorInformation": null,
  "statusReasons": null,
  "purposeCode": null,
  "categoryPurposeCode": null,
  "possibleActions": []
}
```

### Error responses

| Status | Meaning |
| --- | --- |
| `400` | Bad request / validation (e.g. invalid UUID for `payment-id`) |
| `403` | Forbidden |
| `404` | Not found |
| `500` | Internal server error |

`400` example shape:

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "payment-id": [
      "The value '{invalid-payment-id}' is not valid."
    ]
  }
}
```

---

## GET `/api/v1/payments/singles/{payment-id}/status`

Get status of a payment only (no full payment body).

- **Source:** `reference/get_api-v1-payments-singles-payment-id-status.md` — `updatedAt: 2026-06-08T06:01:06.000Z`
- **URL:** https://docs.bankingcircleconnect.com/reference/get_api-v1-payments-singles-payment-id-status
- **Also in guide:** `quick-start.md` (sandbox)

### Parameters

| Name | In | Required | Type | Description |
| --- | --- | --- | --- | --- |
| `payment-id` | path | yes | string (uuid) | Payment identifier (`Payment.paymentId`) |

### Request example

```http
GET https://www.bankingcircleconnect.com/api/v1/payments/singles/{payment-id}/status
Authorization: Bearer {access-token}
Accept: application/json
```

```bash
curl --request GET \
  --url 'https://sandbox.bankingcircleconnect.com/api/v1/payments/singles/{payment-id}/status' \
  --header 'Accept: application/json' \
  --header 'Authorization: Bearer {access-token}'
```

### Response example (`200`)

OpenAPI example:

```json
{
  "status": "PendingProcessing"
}
```

Quick-start success example after processing:

```json
{
  "status": "Processed"
}
```

Initiate response (context from quick-start) returns `paymentId` + `status`:

```json
{
  "paymentId": "{payment-id}",
  "status": "PendingProcessing"
}
```

### Error responses

| Status | Meaning |
| --- | --- |
| `403` | Forbidden |
| `404` | Not found |
| `500` | Internal server error |

---

## GET `/api/v1/payments/singles/transactionreference/{transactionreference}`

Get details of a payment by transaction reference number (same `Payment` payload as GET by payment id).

- **Source:** `reference/get_api-v1-payments-singles-transactionreference-transactionreference.md` — `updatedAt: 2026-06-08T06:01:06.000Z`
- **URL:** https://docs.bankingcircleconnect.com/reference/get_api-v1-payments-singles-transactionreference-transactionreference

### Parameters

| Name | In | Required | Type | Description |
| --- | --- | --- | --- | --- |
| `transactionreference` | path | yes | string | Transaction reference / contract reference (`Payment.TransactionReference`) |

### Request example

```http
GET https://www.bankingcircleconnect.com/api/v1/payments/singles/transactionreference/{transaction-reference}
Authorization: Bearer {access-token}
Accept: application/json
```

```bash
curl --request GET \
  --url 'https://www.bankingcircleconnect.com/api/v1/payments/singles/transactionreference/{transaction-reference}' \
  --header 'Accept: application/json' \
  --header 'Authorization: Bearer {access-token}'
```

### Response example (`200` — Payment)

Same shape as [GET by payment id](#get-apiv1paymentssinglespayment-id). Illustrative redacted sample:

```json
{
  "paymentId": "{payment-id}",
  "transactionReference": "{transaction-reference}",
  "concurrencyToken": "{concurrency-token}",
  "classification": "Outgoing",
  "subClassification": "Internal",
  "isTraceable": true,
  "status": "Processed",
  "processedTimestamp": "2024-06-14T11:17:42+00:00",
  "latestStatusChangedTimestamp": "2024-06-14T11:17:42+00:00",
  "booked": true,
  "reversalBooked": null,
  "return": null,
  "paymentRail": "Internal",
  "errors": [],
  "lastChangedTimestamp": "2024-06-14T11:17:42+00:00",
  "debtorInformation": {
    "paymentBulkId": null,
    "accountId": "{account-id}",
    "account": {
      "accountNumber": "{account-number}",
      "accountIban": "{account-iban}",
      "account": "{account-iban}",
      "financialInstitution": "{bic}",
      "country": null
    },
    "vibanId": null,
    "viban": null,
    "instructedDate": "2024-06-14T00:00:00+00:00",
    "debitAmount": {
      "currency": "DKK",
      "amount": 5.75
    },
    "debitValueDate": "2024-06-14T00:00:00+00:00",
    "fxRate": null,
    "instruction": {
      "debtorAccount": {
        "account": "{account-iban}"
      },
      "debtorReference": "TEST REFERENCE 1",
      "currencyOfTransfer": "DKK",
      "amount": {
        "currency": "DKK",
        "amount": 5.75
      },
      "requestedExecutionDate": "2024-06-14T00:00:00+00:00",
      "chargeBearer": "SHA",
      "creditorAccount": {
        "account": "{creditor-iban}",
        "financialInstitution": "{bic}",
        "country": "DK"
      },
      "creditorName": "CREDITOR NAME",
      "instructedChargeBearer": "SHA"
    }
  },
  "transfer": {
    "amount": {
      "currency": "DKK",
      "amount": 5.75
    },
    "valueDate": "2024-06-14T00:00:00+00:00",
    "chargeBearer": "SHA",
    "creditorName": "CREDITOR NAME",
    "instructedChargeBearer": "SHA"
  },
  "creditorInformation": null,
  "statusReasons": null,
  "purposeCode": null,
  "categoryPurposeCode": null,
  "possibleActions": []
}
```

### Error responses

| Status | Meaning |
| --- | --- |
| `400` | Bad request / validation |
| `403` | Forbidden |
| `404` | Not found |
| `500` | Internal server error |

---

## Payment status meanings

**Source:** Connect payment-status guide  
**URL:** https://docs.bankingcircleconnect.com/docs/payment-status  
**updatedAt:** `2025-09-17T15:19:55.000Z`

OpenAPI `PaymentStatus` enum values: `Unknown`, `ScaExpired`, `ScaFailed`, `ScaPending`, `PendingApproval`, `MissingFunding`, `PendingProcessing`, `Hold`, `PendingCancellation`, `PendingCancellationApproval`, `DeclinedByApprover`, `Rejected`, `Cancelled`, `Processed`, `Reversed`, `ScaDeclined`, `Approved`.

Statuses related to approvals depend on company setup and user permissions.

| Status | Meaning |
| --- | --- |
| Processed | Successfully processed; final. Passed validation and AML; sent to recipient. |
| Pending Processing | Waiting to be processed or in progress. |
| Pending Approval | Waiting for another user in the company to approve. |
| Pending Cancellation | Waiting to be cancelled by another user. |
| Pending Cancellation approval | Cancellation requested; awaiting additional approval. Then moves to Cancelled. |
| Missing Funding | Insufficient funds. Auto-processes if funded within two days; otherwise rejected. |
| Hold | Temporarily paused (manual review, confirmation, compliance/AML). May move to Processed, Pending Approval, or Rejected. |
| Declined | Declined by an approver in the approval workflow. |
| Rejected | Rejected (e.g. Missing Funding for two days, or invalid fields). |
| Cancelled | Cancelled by the company. |
| Reversed | Rejected by the payment scheme after Processed (rare). |
| Returned | **Not an actual system state.** Original stays Processed; return appears as incoming Processed. Identify via remittance / `return` flag. |
| SCA Expired | Strong Customer Authentication session expired. |
| SCA Failed | SCA attempt unsuccessful. |
| SCA Pending | Awaiting SCA. |
| SCA Declined | SCA declined by the user. |

### Outgoing-payments lifecycle subset

**Source:** `outgoing-payments.md` — `updatedAt: 2025-10-14T09:46:37.000Z`

Common API statuses emphasized for outgoing flows: `PendingApproval`, `PendingProcessing`, `Processed`, `MissingFunding`, `Rejected`, `Cancelled`, `Reversed`. Direct debits appear as `PendingProcessing` the day before due.

Returned payments: original remains `Processed`; `IncomingPaymentProcessed` webhook / GET payments may show `"return": true`.

### Other GET paths mentioned in guides

| Method | Path | Where mentioned |
| --- | --- | --- |
| GET | `/api/v1/payments/singles` | `practical-guide-reconciliation-using-webhooks.md` (returns / `return` flag) |
