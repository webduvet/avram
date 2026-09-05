# Reports adjacent to Oversight

Sources: `banking-circle-reports.md`, `transaction-extract.md`, cross-referenced from Oversight payment handoff guidance.

These are **not** Oversight `/oversight/v1` endpoints. They are used **with** Oversight after (or instead of) real-time settlement webhooks: Oversight surfaces `banking_circle_api_response.paymentId` at handoff; settlement confirmation then comes from Banking Circle or, for on-platform card/float programmes, day-end extracts.

## Banking Circle reports (post-handoff)

Audience: partners on Banking Circle Connect (mTLS). Official hub: [Banking Circle Connect](https://docs.bankingcircleconnect.com/).

Layered model (do not expect one export to contain everything):

| Layer | BC source | Answers |
|---|---|---|
| Booking ledger | Reconciliation report | What booked, when, amounts, `paymentId` |
| Statement | Account activity report | Per-account lines, balance, counterparty |
| Payment object | `GET /api/v1/payments/singles/{paymentId}` | Full payment object, status, rail |

Common endpoints noted in local extract / KEY-EXTRACTS:

- `GET /api/v1/payments/singles/{paymentId}` — ~6 months retention observed (404 for older IDs; tenant variation)  
- `GET /api/v1/reports/reconciliation-report` (+ paged / intraday variants)  
- `POST /api/v2/reports/requests/reconciliation-report` — async large history  
- Account activity / rejection reports; accounts, balances, virtual accounts  
- Traces / cases; async sanctions-screening report  

Correlate BC `paymentId` from Oversight’s final webhook. Identifier families (`*Id` UUIDs vs string references) must not be mixed — see full `banking-circle-reports.md` for field-name pitfalls.

## Transaction extract (on-platform, not Oversight)

Day-end JSON batch files (card + float transactions) via B4B SFTP, generated ~1am UK time. Intended to duplicate webhook notification data for reconciliation. Setup needs source IP(s) and an SSH public key. This is the **on-platform** programme extract path, not Oversight payment settlement.

## Practical split

| Need | Use |
|---|---|
| Regulatory / pre-settlement status | Oversight `callback_url` or `GET /payments…` |
| Real-time settlement after handoff | BC webhooks on `paymentId` |
| Batch settlement / booking | BC reconciliation / account-activity reports |
| Card/float day-end reconciliation | On-platform transaction extract (SFTP) |
