# Banking Circle

B2B infrastructure bank: accounts, local and cross-border payments, FX, and treasury, sold as white-label rails to FIs, PSPs, and funds. **Not retail** — no personal accounts, no direct-to-consumer.

Contracting core is **Banking Circle S.A.**, a Luxembourg **credit institution** (not an EMI). Legal facts belong on the official [regulatory information](https://www.bankingcircle.com/regulatory-information/) and [disclaimers](https://www.bankingcircle.com/disclaimers/) pages. A CSSF fraud warning about **“Circle Group S.A.”** is a possible **name collision** — it is not proof of this firm’s licence.

## Licence map (short)

| Entity | Role |
|---|---|
| Banking Circle S.A. (LU) | Credit institution, home supervisor CSSF. Contracting core. |
| EU/EEA branches | Denmark, Germany, Sweden, Norway |
| UK | Third-country branch, PRA-authorised / FCA+PRA-regulated, FRN 848617 (live FCA register body not retrieved) |
| Banking Circle (Liechtenstein) AG | FMA; CHF local clearing |
| BC Payments Pte. Ltd. (SG) | MAS Major Payment Institution |
| Australian Settlements Limited t/a Banking Circle | APRA ADI; wholesale only; AU obligations not guaranteed by BCSA |
| BCUS, Inc. | Connecticut uninsured bank; correspondent/sister, **not** a BCSA subsidiary |

Poland / Czech Republic branches appear in Apr 2026 press “About” and **not** on the regulatory/disclaimers pages.

## Notes

- [Integrator baseline](integrator-baseline.md) — product map, API hosts, sourced findings, open questions (2026-08-29)
- [Payment confirmation](payment-confirmation.md) — webhooks (fast) vs poll/recon (ultimate); what Processed / Rejected mean
- [Webhooks](webhooks/index.md) — Connect webhook setup and troubleshooting
- [B4B Oversight payment tracking](../b4b-payments/oversight-payment-tracking.md) — Oversight handoff vs BC settlement
