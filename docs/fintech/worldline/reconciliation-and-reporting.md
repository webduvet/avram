# Reconciliation and reporting

Operational recon for Worldline Connect / Global Collect: online confirmation plus daily offline reports, with SFT as the primary file path used in-stack.

Lead FAQ: [Reconciliation and reporting](https://docs.connect.worldline-solutions.com/support/faq/connect/reconciliation-and-reporting)

Parent: [Worldline](index.md) · related: [Acquiring](acquiring.md)

## Two-layer model

Source: [Reporting overview](https://docs.connect.worldline-solutions.com/reporting/)

1. **Online reporting** — each online transaction is confirmed online with an approval or rejection message.
2. **Daily offline operational reports** — daily payment reports overview payments processed that day, broken down to transaction level.

Online status (API / webhooks / console) is not a substitute for WX-based settlement recon.

## Standard reports (current)

| Report | Purpose | Formats | Cadence | Delivery |
|---|---|---|---|---|
| **WX (Operational)** | Processed orders, collected transactions, totals per currency; separate file per merchant ID / Account ID / Contract ID; used to update order management from platform activity | XML, CSV, ASCII | 7 days/week; uploaded before **00:00 CET** | SFT `/out` |
| **Financial Report** | Financial sections; remittance content depends on daily vs weekly remittance setup. Operational overview reflects daily processing/collections; remittance sections follow the agreed remit cut-off (e.g. weekly remit → remittance overview data on Friday) | PDF, CSV, XML UTF8 | Monday–Friday | SFT `/out` |

Source: [Reporting overview](https://docs.connect.worldline-solutions.com/reporting/)

**Decommissioned (legacy GlobalCollect):** Collection reports and Financial Statements are **not provided anymore**. Historical data on request. Daily email delivery of reports is **not** offered; files go to the merchant’s SFT directory (or download of standard WX + Financial Report from Payment Console). FAQ: [Reconciliation and reporting](https://docs.connect.worldline-solutions.com/support/faq/connect/reconciliation-and-reporting).

Full technical WX field specifications: contact merchant services (not fully public on the docs site).

## How files are retrieved

### Primary — Managed File Transfer / SFT (SFTP)

Source: [Secure file transfer](https://docs.connect.worldline-solutions.com/reporting/secure-file-transfer)

| | |
|---|---|
| Host | `prod.mft.worldline-solutions.com` |
| Port | `22` |
| Receive | `/out` |
| Send | `/in` |

Retrieval methods documented:

1. **MFT Web Client** — Files → `out`
2. **WinSCP** (or equivalent SFTP client) with credentials from the implementation manager
3. **Automated script** / SSH key upload per the access manual provided at onboarding

**Documented (Worldline):** WX is uploaded **daily**, 7 days/week, to the designated SFT environment **before 00:00 CET**. That is a Worldline **daily upload**, not a documented twice-daily push.

**Observed (internal practice):** the stack currently **polls SFTP about twice per day** to pick up files. That poll cadence is integrator behaviour. Do **not** treat ~2×/day as official Worldline delivery frequency.

### Other means

1. **Payment Console** — download of standard WX (Operational) and Financial Report (FAQ).
2. **Insights** portal — merchant reporting for Global Collect; Financials dashboard for settlement navigation; CSV export of transactions; role/permission controls. Source: [Insights](https://docs.connect.worldline-solutions.com/reporting/insights).
3. **Online reporting / API + webhooks** — near-real-time status. Complements but does **not** replace WX settlement recon.

### Multiple SFT directories

FAQ: the same reports **cannot** be duplicated to two separate SFT directories. On request, configuration can deliver **different report types** to different directories (FAQ wording still refers to daily / collection / financial directories — note **collection reports are decommissioned**; confirm current split options with merchant services).

### Sandbox

Sandbox **neither** generates reports **nor** provides the Payment Console feature. Reports apply to GlobalCollect pre-production and live. FAQ: [Reconciliation and reporting](https://docs.connect.worldline-solutions.com/support/faq/connect/reconciliation-and-reporting).

## File shape / naming

- Formats as in the table above. Full field specs via merchant services.
- **Secondary (third-party integrator notes — not Worldline primary docs):**
  - IXOPAY adapter docs: production WX files often prefixed `wx1.`; test/staging `wxt.` when test mode is enabled on the SFTP fetch.
  - Zuora Global Collect reporting: older `wr` / `wr1` files **deprecated** in favour of **`wx1`**; use `MERCHANTREFERENCENUMBER` for payment matching / reconciliation (Zuora maps `Payment.GatewayOrderId` or a `PaymentNumber-TenantID` composite into that field).
- **Illustration of WX operational content (official Troy product reporting):** daily WX carries payments / refunds / chargebacks with transaction type codes such as `XIP` (Captured Payment), `+IP` (Payment Received), `-IP` (Correction of Payment), `-RF` / `+RF` (refund / correction), `-RI` / `+RI` (reversal/chargeback / correction). Source: [Troy reporting (GlobalCollect)](https://docs.connect.worldline-solutions.com/payment-product/troy/reporting/globalcollect).

## Controls / configuration

| Control | Notes |
|---|---|
| Remittance schedule (daily vs weekly) | Changes which Financial Report remittance sections populate and when |
| Report formats | WX: XML / CSV / ASCII; Financial: PDF / CSV / XML UTF8 — via merchant services / onboarding setup |
| SFT credentials / SSH keys | Implementation manager + onboarding access manual |
| Report type → SFT directory split | On request; same report cannot be mirrored to two dirs (FAQ) |
| Insights roles / permissions | Portal access control; audit trail of user activity |
| Full WX technical specs | Merchant services |

## Sources

Official Worldline / Connect:

- [Reconciliation and reporting FAQ](https://docs.connect.worldline-solutions.com/support/faq/connect/reconciliation-and-reporting)
- [Reporting overview](https://docs.connect.worldline-solutions.com/reporting/)
- [Secure file transfer](https://docs.connect.worldline-solutions.com/reporting/secure-file-transfer)
- [Insights](https://docs.connect.worldline-solutions.com/reporting/insights)
- [Troy reporting (GlobalCollect) — WX type codes example](https://docs.connect.worldline-solutions.com/payment-product/troy/reporting/globalcollect)

Secondary integrator docs (naming / matching only):

- [IXOPAY — Ingenico / Worldline / Ogone adapter](https://documentation.ixopay.com/manual/adapters/ingenico-direct) (`wx1.` / `wxt.`)
- [Zuora — Worldline Global Collect reporting](https://docs.zuora.com/en/zuora-payments/manage-payment-gateway-integrations-and-payment-methods/set-up-payment-gateway-integrations/worldline-global-collect-payment-gateway/worldline-global-collect-reporting) (`wx1` vs deprecated `wr`; `MERCHANTREFERENCENUMBER`)
