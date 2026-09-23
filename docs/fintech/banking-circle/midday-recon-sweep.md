# Midday recon sweep (outbound payout confirm)

Integrator pattern for confirming same-day **outbound** payouts when webhooks are missing, late, or not subscribed. Complements [Payment confirmation](payment-confirmation.md) and [Payment lifecycle](payment-lifecycle.md). Field glossary for recon rows: [Documentation extract — reconciliation](documentation-extract/reconciliation.md) and the official [Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report) guide (`return`, `processedTimestamp`, remittance lines, etc.).

**Observed / integrator pattern — cadence:** issue window about **08:00–11:00** local; must be able to send by **16:00** local. Do **not** wait for the 19:00 CET creation bucket. First bulk pull ~midday (or ~one hour after last issue); optional second sweep before 16:00 for leftovers still `pendingProcessing`.

Official stance: do **not** use webhooks as the sole reconciliation process ([Practical guide](https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks), [Webhooks](https://docs.bankingcircleconnect.com/docs/webhooks)). Prefer recon reports + status for completeness.

## Primary success bulk — intraday recon paged

**Preferred endpoint:** `GET /api/v1/reports/intraday-reconciliation-paged-report`

OpenAPI: https://docs.bankingcircleconnect.com/reference/get_api-v1-reports-intraday-reconciliation-paged-report  
(Guide prose still names `GET /api/v1/reports/reconciliation-intraday-report`; that reference slug has been observed **404** live. Local mirror retains a large file for the old slug, but the **working** OpenAPI path for a creation-time midday window is the **paged** endpoint above.)

### Why this endpoint

- Same-day monitoring with a **creation-time** window (`FromCreatedAt` / `ToCreatedAt`), not only calendar transaction date.
- Prefer over non-intraday `reconciliation-paged-report` (single `TransactionDate`, no created-at window) and over **account-activity** reports (statement/booking by transaction date — not the right primary for outbound status confirm).

### Required query params (OpenAPI)

| Param | Required | Notes |
| --- | --- | --- |
| `FromTransactionDate` | yes | `YYYY-MM-DD` |
| `ToTransactionDate` | yes | `YYYY-MM-DD`. After **19:00 CET**, OpenAPI: set to **next business day** to obtain payments after EOD |
| `FromCreatedAt` | yes | Creation lower bound (`YYYY-MM-DDTHH:MM:SS.0000000Z`) |
| `ToCreatedAt` | yes | Creation upper bound |
| `PageNumber` | yes | 1..n |
| `PageSize` | yes | Positive; above internal max reset (e.g. 5000) |

Optional: `PropertiesIncluded` / `PropertiesExcluded` (mutually exclusive with deprecated Include/Exclude aliases), `AccountId` (comma-separated account GUIDs). If `PropertiesIncluded` is set, it **replaces** defaults — include needed defaults plus extras such as `PaymentId`, `ProcessedTimestamp`, `LatestStatusChangedTimestamp`, `Return`.

### Status population (recon family)

All Connect reconciliation report types include payments with status **`processed`** and **`pendingProcessing`** only — **not** `Rejected` or `MissingFunding` ([Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report)).

**Presence on the report ≠ `Processed`.** Rows may still be `pendingProcessing`. Use `processedTimestamp` (optional property) and/or singles status.

### 19:00 CET creation bucket

Guide: payments after **19:00 CET** appear in the **next business day’s** report ([Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report)). OpenAPI for the paged intraday path: last available report for `transaction date = current calendar date` is at 19:00 (EOD); bump `ToTransactionDate` for post-19:00 CET activity. A midday sweep before EOD is unaffected by waiting for 19:00.

### Matcher rules

**Observed / integrator pattern** (not a verbatim Connect algorithm):

- Intersect report rows with the integrator’s **own issued** `paymentId` set for the run.
- Prefer outbound lines: e.g. `creditDebitIndicator` = `DBIT` where present.
- Treat `return` = `true` as an **incoming return** row (separate payment), not confirmation of the original outbound; typical outbound confirms have `return` **`null`** ([Reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report) field definition). See [Reverse vs Return](reverse-vs-return.md).
- Confirm success via non-null `processedTimestamp` and/or `GET .../status` = `Processed`; leave pure `pendingProcessing` for a later pass.
- Match also on `userReferenceNumber` / remittance (`paymentDetails*`) when `paymentId` was not requested on the report.

## Bulk fail / stuck — Rejection Report

For payments that never appear on recon (rejects / missing funding / stuck pending).

### Sync

`GET /api/v1/reports/rejection-report`  
OpenAPI: https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-reports-rejection-report

| Param | Required | Notes |
| --- | --- | --- |
| `TransactionDate` | **yes** | `YYYY-MM-DD` |
| `IncludeReceived` | no | Include pending processing |
| `IncludeMissingFunds` | no | Include insufficient funds |
| `IncludeReversals` | no | Direct Debit / SEPA DD reversals only (OpenAPI wording) |
| `ExcludeBooked` | no | Default **false** |

**Default population** ([Rejection report](https://docs.bankingcircleconnect.com/docs/rejection-report)): **Missing funding**, **Pending processing**, **Rejected**. Empty if all payments processed correctly.

**MissingFunding window on the report:** OpenAPI/guide — payments in Missing funding remain on the report for **2 days**; if still unfunded, they end as rejections.

### Async

Guide documents `POST /api/v2/reports/requests/rejection-report` ([Rejection report](https://docs.bankingcircleconnect.com/docs/rejection-report)). Local OpenAPI mirror for that POST was **not** present at authoring time — do not invent body fields; follow Connect reports-on-request flow when using async.

## Leftovers — payment status

`GET /api/v1/payments/singles/{payment-id}/status`  
OpenAPI: https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-payments-singles-payment-id-status

Path: `payment-id` (uuid). Returns status only (example: `PendingProcessing`). Use for pending rows missing from recon, suspected `Rejected` / `MissingFunding` / `Hold`, or still pending after the bulk pull.

Optional list: `GET /api/v1/payments/singles` (filter by status / dates) — see [Payment lifecycle](payment-lifecycle.md) / Connect [Payment lifecycle](https://docs.bankingcircleconnect.com/docs/payment-lifecycle). Prefer targeted status GETs for leftovers over inventing list filters.

## Sweep recipe

**Observed / integrator pattern:**

1. **Webhooks** (if subscribed) — fast path enqueue settlement on `OutgoingPaymentProcessed` / `OutgoingPaymentRejected` / `Reversed` (see [Payment lifecycle](payment-lifecycle.md)).
2. **Intraday recon paged** — primary bulk for same-day created outbound; client-side match; confirm Processed via `processedTimestamp` / status.
3. **Rejection report** — bulk view of Missing funding / Pending processing / Rejected for the transaction date.
4. **GET status** — leftovers still unknown or not on either report.
5. **Second sweep before 16:00** local if needed (slow rails may still be `pendingProcessing` shortly after issue).

Same settlement-job semantics as [Payment confirmation](payment-confirmation.md): idempotent on `paymentId` + status; worker is the only ledger writer.

## Sources

- https://docs.bankingcircleconnect.com/docs/reconciliation-report
- https://docs.bankingcircleconnect.com/docs/rejection-report
- https://docs.bankingcircleconnect.com/docs/practical-guide-reconciliation-using-webhooks
- https://docs.bankingcircleconnect.com/docs/webhooks
- https://docs.bankingcircleconnect.com/reference/get_api-v1-reports-intraday-reconciliation-paged-report
- https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-reports-rejection-report
- https://docs.bankingcircleconnect.com/bankingcircle/reference/get_api-v1-payments-singles-payment-id-status
- https://docs.bankingcircleconnect.com/docs/outgoing-payments
- https://docs.bankingcircleconnect.com/docs/payment-status
