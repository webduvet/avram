# Accounts / balance

Account listing and embedded balances from the Connect quick-start guide. A dedicated balance-by-id OpenAPI page is **still missing** from the local Connect reference mirror.

**Guide source:** Connect quick-start guide — `updatedAt: 2025-09-17T15:24:54.000Z`  
**URL:** https://docs.bankingcircleconnect.com/docs/quick-start

---

## GET `/api/v1/accounts`

List company accounts (includes `balances` with `CurrentBalance` and related fields). OpenAPI for this path is **guide-only locally** (no `get_api-v1-accounts*.md` under `reference/`).

### Headers

```
Accept: application/json
Authorization: Bearer {access-token}
```

### Request example

```http
GET https://sandbox.bankingcircleconnect.com/api/v1/accounts
Accept: application/json
Authorization: Bearer {access-token}
```

```bash
curl --request GET \
  --url 'https://sandbox.bankingcircleconnect.com/api/v1/accounts' \
  --header 'Accept: application/json' \
  --header 'Authorization: Bearer {access-token}'
```

### Response example (balances sample from quick-start)

IDs redacted to placeholders. Balance entry type shown is `CurrentBalance`.

```json
{
  "result": [
    {
      "accountId": "{account-id}",
      "accountDescription": "string",
      "accountIdentifiers": [
        {
          "account": "string",
          "financialInstitution": "string",
          "country": "string"
        }
      ],
      "ibans": [
        "string"
      ],
      "status": "Closed",
      "currency": "string",
      "openingDate": "2025-07-08",
      "closingDate": "2025-07-08",
      "ownedByCompanyId": "{company-id}",
      "ownedByCompanyName": "string",
      "ownedByCompanyNumber": "string",
      "protectionType": "None",
      "balances": [
        {
          "type": "CurrentBalance",
          "currency": "string",
          "beginOfDayAmount": 0,
          "financialDate": "2025-07-08",
          "intraDayAmount": 0,
          "lastTransactionTimestamp": "2025-07-08T10:08:12.368Z",
          "blockedBalanceAmount": 0
        }
      ],
      "friendlyName": "string",
      "netInterestRate": 0,
      "interestCalcMethod": "string",
      "overdraftRate": 0,
      "overdraftRateCalcMethod": "string"
    }
  ],
  "pageInfo": {
    "currentPage": 0,
    "pageSize": 0,
    "rowCount": 0
  }
}
```

### Balance fields (from sample)

| Field | Notes |
| --- | --- |
| `type` | e.g. `CurrentBalance` |
| `currency` | Balance currency |
| `beginOfDayAmount` | Begin-of-day amount |
| `financialDate` | Financial / booking date |
| `intraDayAmount` | Intraday amount |
| `lastTransactionTimestamp` | Last transaction time |
| `blockedBalanceAmount` | Blocked amount |

### Using the response for payments / webhooks

- Debtor/creditor account values for payment initiation come from `accountIdentifiers.account` (IBAN or local account number). If local account number, also use `accountIdentifiers.financialInstitution`.
- Webhook subscription `targetIds`: company GUID → `ownedByCompanyId`, account GUID → `accountId` (see notification self-service docs).

---

## Dedicated balance-by-id OpenAPI — still missing

No local reference markdown for a dedicated balance endpoint such as `GET /api/v1/accounts/{id}/balances` (or equivalent). Confirm against live docs index / llms.txt when fetching:

- https://docs.bankingcircleconnect.com (search accounts / balances under Reference)

Until then, balances are documented only as embedded on `GET /api/v1/accounts` via the quick-start sample above.

---

## Other account-related paths in local guides

| Path / topic | Source | Notes |
| --- | --- | --- |
| `GET /api/v1/accounts` | `quick-start.md`, `post_api-v1-notificationselfservice-subscriptionevent.md` | List accounts; `accountId` / `ownedByCompanyId` for webhook targets |
| Agency Banking whitelist simulation | `simulating-account-services.md` | POST/GET whitelist; not a balance API |
| Account aliases (PayID) | `simulating-account-services.md` | PUT/GET alias workflows |
| Account Holder Verification | `account-holder-verification.md` | Verification API / webhook `AccountHolderVerification` |
| Available vs booked balance (conceptual) | `practical-guide-reconciliation-using-webhooks.md` | Conceptual; not an HTTP endpoint |
