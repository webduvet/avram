# Reconciliation reports

Extract from the Connect reconciliation report guide. Local OpenAPI reference files for these report endpoints are **not** present in the local Connect reference mirror; parameters below come from the guide text.

**Source:** Connect reconciliation-report guide  
**URL:** https://docs.bankingcircleconnect.com/docs/reconciliation-report  
**updatedAt:** `2026-05-22T10:40:05.000Z`

Webhooks are not recommended as a replacement for reconciliation.

## Sync limit (50,000) and async path

| Mode | Endpoint | Volume |
| --- | --- | --- |
| **Synchronous** | `GET /api/v1/reports/reconciliation-report` | **Up to 50,000 payments per report** |
| **Asynchronous (reports-on-request)** | `POST /api/v2/reports/requests/reconciliation-report` | Larger volumes beyond the sync limit |
| **Paginated (sync alternative)** | `GET /api/v1/reports/reconciliation-paged-report` | Single-day for volumes **> 50,000**; customizable page sizes; **no date range** |

Quoted from the guide:

> Up to 50,000 payments per report using the synchronous endpoint

> Also available with larger volume via the asynchronous reports-on-request endpoint (`POST /api/v2/reports/requests/reconciliation-report`)

> Single-day reconciliation for large volumes (>50,000 payments)

**No curl or full HTTP request examples** appear in `reconciliation-report.md`; only endpoint paths, field tables, and the volume guidance above. OpenAPI request/response samples require a live fetch of the reference pages listed below.

## Report types

| Report | Method / path | Notes |
| --- | --- | --- |
| Standard | `GET /api/v1/reports/reconciliation-report` | Historical, multi-day. **≤ 50,000 payments** on sync; larger via async POST above. |
| Paginated | `GET /api/v1/reports/reconciliation-paged-report` | Single-day for **> 50,000**; customizable page sizes; no date range. |
| Intraday | `GET /api/v1/reports/reconciliation-intraday-report` | Same-day monitoring with time-range filtering; tracks by creation time. **Payments after 19:00 CET appear in the next business day’s report.** |

All three include payments with status `processed` and `pendingProcessing`. Formats: **JSON** and **CSV**.

Reference OpenAPI (needs live fetch — not in local mirror):

- https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-reports-reconciliation-report
- https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-reports-reconciliation-intraday-report
- https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-reports-reconciliation-paged-report
- https://docs.bankingcircleconnect.com/bankingcircle/reference/post_api-v2-reports-requests-reconciliation-report

### 19:00 CET cutoff (quoted)

> Note: Payments after 19:00 CET will appear in the next business day's report

## Parameters mentioned in the guide

Full query schemas are not in the local mirror. The guide documents:

| Parameter | Usage |
| --- | --- |
| `PropertiesIncluded` | Request optional fields to include |
| `PropertiesExcluded` | Exclude fields from the report |
| Format | JSON or CSV (field naming differs — see below) |

Account / date / time query parameters for the GET endpoints are expected in the live OpenAPI pages listed above.

## Field naming

- JSON: camelCase (e.g. `pIdChannelUser`, `pTxnDate`)
- CSV: capitals and underscores (e.g. `P_ID_CHANNELUSER`, `P_TXNDATE`)

## Mandatory fields

Always included by default:

| Field | Type (guide) | Example | Description |
| --- | --- | --- | --- |
| pIdChannelUser | Varchar | user@example.com | User ID of requesting client |
| pTxnDate | Date | 2021-10-27T00:00:00.000+00:00 | When payment was instructed in BC Connect (not processing date) |
| reportDate | Date (DDMMYYYY) | 29102021 | Report generation date |
| customerId | Number | `{customer-id}` | Customer ID of the account |
| account | Varchar | DKXXXX0000000XXXXX | IBAN |
| accountCurrency | Text (3) | EUR | ISO currency of account |
| debitAmount | Number, 2 dp | 22.26 | Debited amount if DBIT (negative on debit reversal) |
| creditAmount | Number, 2 dp | 10.89 | Credited amount if CRDT (negative on credit reversal) |
| valueDate | Date | 27/10/2021 | Value date for posting |
| instructedAmount | Number | 22.26 | DBIT only — original currency after FX |
| instructedAmountCurrency | Text (3) | EUR | Instructed currency |
| transactionAmount | Number | 18.78 | Transaction amount |
| transactionAmountCurrency | Text (3) | GBP | Transaction currency |
| exchangeRate | Number | 0.84367 | FX rate |
| debtorBankCode | Varchar | | Debtor bank (CRDT) |
| debtorAccount | Varchar | | Debtor account |
| debtorLine1 … debtorLine4 | Varchar | | Debtor name/address lines |
| beneficiaryBankCode | Varchar | | Beneficiary bank (DBIT) |
| beneficiaryAccount | Varchar | | Beneficiary account |
| beneficiaryLine1 … beneficiaryLine4 | Varchar | | Beneficiary lines |
| paymentReferenceNumber | Varchar | | BC Connect reference (unique for processed txns) |
| fileReferenceNumber | Varchar | | Bulk id when from bulk file |
| userReferenceNumber | Varchar | | DebtorReference from instruction |
| paymentDetails1 … paymentDetails4 | Varchar | | Remittance / details (line 1 propagated to beneficiary) |
| creditDebitIndicator | DBIT/CRDT | DBIT | Credit or debit |
| bankTrnsCodeDomain | Varchar | PMNT | Bank transaction code domain |
| bankTrnsCodeFamily | Varchar | ICDT | e.g. ICDT issued / RCDT incoming |

## Additional optional fields

Included when requested via `PropertiesIncluded` (or omitted via `PropertiesExcluded`):

| Field | Notes |
| --- | --- |
| bankTrnsCodeSubFamily | DMCT / XBCT / RRTN |
| paymentId | UUID matching API payment id |
| lastChangedTimestamp | Last change in processing system |
| createdAt | Creation timestamp |
| additionalRemittanceInformation1…5 | Extra remittance; on returns may hold return indicator / reason / original reference |
| debtorAgentBankCode | BIC of debtor’s institution |
| ultimateDebtorLine1…4 | Ultimate debtor name/address (debits) |
| ultimateDebtorAccount | Ultimate debtor account (debits) |
| ultimateBeneficiaryAccount | Ultimate beneficiary account (credits) |
| statusReasonCode | Reversal reason code |
| statusReasonDescription | Reversal reason text |
| instructedChargeBearer | Charge bearer as instructed |
| chargeBearer | Actual charge bearer used |
| paymentRail | e.g. SEPA (SCT) |
| return | `true` if incoming return, else `null` (not for incoming SEPA DD) |
| directDebitMandateId | SEPA DD mandate id |
| processedTimestamp | When payment processed |
| latestStatusChangedTimestamp | Latest status change |
| clientOrderId | FX trade client order id (FX only) |

## Example field values (from guide; IDs redacted)

Illustrative JSON-style fragment using guide examples with redaction (not a full API response — the guide has no curl/response sample):

```json
{
  "pIdChannelUser": "user@example.com",
  "pTxnDate": "2021-10-27T00:00:00.000+00:00",
  "reportDate": "29102021",
  "customerId": "{customer-id}",
  "account": "DKXXXX0000000XXXXX",
  "accountCurrency": "EUR",
  "debitAmount": 22.26,
  "creditAmount": null,
  "valueDate": "27/10/2021",
  "instructedAmount": 22.26,
  "instructedAmountCurrency": "EUR",
  "transactionAmount": 18.78,
  "transactionAmountCurrency": "GBP",
  "exchangeRate": 0.84367,
  "creditDebitIndicator": "DBIT",
  "paymentReferenceNumber": "{payment-reference}",
  "paymentId": "{payment-id}",
  "return": null
}
```

No full HTTP request/response OpenAPI examples were available in the local mirror for these report endpoints.
