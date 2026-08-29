# Integrator baseline

Research snapshot as of 2026-08-29. Primary sources unless flagged. Webhooks are filed separately: [Setup webhooks](webhooks/setup-webhooks.md), [Troubleshooting](webhooks/troubleshooting.md). Connect docs portal: [webhooks](https://docs.bankingcircleconnect.com/docs/webhooks) (login-walled).

**Method:** official product/legal pages on bankingcircle.com (and AU/SG/LI sites), CSSF/MAS/FCA identifiers, Connect support/ISO guides, and the Connect docs portal. Most of `docs.bankingcircleconnect.com` is password-protected as of this date; API details below are only those confirmed on pages that returned content or on official support PDFs.

!!! warning "Name collision"
    Finding 1 in the source briefing cited a [CSSF warning about “Circle Group S.A.”](https://www.cssf.lu/en/2025/01/warning-concerning-the-fraudulent-activities-carried-out-by-circle-group-s-a/). That is **not** proof of Banking Circle S.A.’s licence. Legal facts below prefer [regulatory information](https://www.bankingcircle.com/regulatory-information/) and [disclaimers](https://www.bankingcircle.com/disclaimers/).

## Product map

| Product | What it does | Who it is for |
|---|---|---|
| **Business / multi-currency accounts** | Hold and move funds; multi-jurisdiction physical accounts; API/Portal/SFTP/SWIFT | PSPs, banks, funds, marketplaces (B2B) |
| **Virtual IBANs / virtual accounts** | Addressable IBAN/account numbers for reconciliation; bulk issue/close; COBO/POBO naming | PSPs, marketplaces, card issuers (funding), FIs |
| **Safeguarded accounts** | Segregated client-money accounts to meet local safeguarding | Regulated clients (PIs/EMIs) |
| **Payments (payouts + collections)** | Local instant/batch/RTGS + SWIFT cross-border; 24 currencies | Same as accounts |
| **SEPA Instant / SCT / T2** | EUR instant, batch credit, high-value RTGS | EUR payers/collectors; agency clients with own BIC |
| **Faster Payments / CHAPS** | GBP instant 24/7; CHAPS RTGS (cut-off 17:45 CET). Direct CHAPS participation described as forthcoming (Jun 2026) | GBP flows; UK agency for FPS |
| **Nordics / AU / CH local** | DKK TIPS/T2/Intradag; SEK RIX/RIX-Inst/Data Clearing; AUD NPP/BECS (BCAU); CHF SIC (BCLI). SIC IP scheduled 2026 | Local-currency clients |
| **SWIFT / correspondent** | Instruct via SWIFT through BC correspondent network; reciprocal Nostro/Vostro | Banks / FIs with BIC |
| **Agency banking** | Indirect SEPA + FPS using **client’s own BIC/IBAN**; no own scheme membership | Regulated SEPA-zone (EUR) or UK (EUR+GBP) entities with SWIFT BIC |
| **BC-NOW** | Book-transfer 24/7 between BCSA accounts | Existing BCSA clients / network counterparties |
| **FX (incl. “white-labelled FX”)** | 24 executable currencies; TOD/TOM/SPOT; 24/7 booking (market hours Sydney Mon 08:00–NY Fri 17:00); pre-agreed margins | PSPs/banks embedding FX for merchants |
| **FlexLiquidity** | Automated physical cash pooling across BC currency accounts | Treasury teams at FIs/PSPs |
| **Fiduciary deposits** | Interest-bearing EUR/GBP/USD deposits, bankruptcy-remote fiduciary estate, accessible in business hours | FIs / corporates / funds |
| **SEPA Direct Debit + RtP** | EUR SDD collections + mandate mgmt; Open Banking RtP EUR/GBP | Payment businesses / merchants via PSPs |
| **EURI (EMT)** | Bank-issued EUR stablecoin, 1:1, redeemable at par | Onboarded CASPs / DA institutions (marketing restricted) |
| **Stablecoin settlement (CASP)** | Fiat↔stablecoin (USDC, USDG, EURI) from core platform; announced 15 Apr 2026 | Digital-asset exchanges, issuers, OTC, web3 platforms |
| **Investment-fund accounts** | Fast multi-ccy accounts for AIFs/GPs/SPVs/feeders; onboarding portal “reserved accounts” | Fund managers / CSPs |
| **Card issuer support** | vIBAN top-up/spend reconciliation — **not** a BC card product | Card issuers / programme managers |
| **Client Portal** | No-code payments, FX, reports, users/approvals; PSD2 SCA | Operations teams (parallel to API) |

## API hosts

Public production/sandbox hosts confirmed in official support / indexed Connect docs:

| Role | URL |
|---|---|
| Auth sandbox | `https://authorizationsandbox.bankingcircleconnect.com/` |
| Auth prod | `https://www.bankingcircleconnect.com/api/v1/authorizations/authorize` |
| API sandbox | `https://sandbox.bankingcircleconnect.com` |
| API prod | `https://www.bankingcircleconnect.com` (ISO guide also says host `bankingcircleconnect.com`) |
| Direct Debit sandbox / prod | `https://sepaexpress-sand-fx.azurewebsites.net/` · `https://sepaexpress-prod-fx.azurewebsites.net/` |
| Client Portal | `https://www.bankingcircleconnect.com/` |

Connect is M2M Bearer JWT. Admin users **cannot** create API users or change API-user certificates — Banking Circle provisions them. Sandbox vs production are separate. Full mTLS + JWT handshake, token TTL, and cert format are **behind login** on Connect docs; third-party SDKs are not treated as primary.

Confirmed resource groups (from pages that returned or were indexed):

- Payments: `POST /api/v1/payments/iso20022/customer-credit-transfer-initiation` (pain.001.001.03 in, pain.002.001.03 out; 201 = accepted for processing, 200 = all rejected; optional `Idempotency-Key` ≤100 chars)
- Agency/correspondent ISO: pacs.008 / pacs.002 (indexed)
- Reports: `GET /api/v1/reports/iso20022/bank-to-customer-end-of-day` (camt.053.001.02); `GET /api/v1/reports/bank-to-customer-account-report` (camt.052 / json / csv)
- Virtual accounts: including `POST /api/v1/virtualAccounts/close/file`; bulk vIBAN ≤1,000 per request
- FX: `POST /api/v1/fx-reports/transactions` plus TOD/TOM/SPOT marketing
- Direct Debit: **separate Azure hosts**, not the main Connect API
- Sandbox payment **final status is simulated** — not production-like

Most of `docs.bankingcircleconnect.com` opened as password-protected. That is current state, not absence of docs.

## Findings (sourced)

1. **Contracting core is Banking Circle S.A., a Luxembourg credit institution (not an EMI).** Société Anonyme at 2 Boulevard de la Foire, L-1528 Luxembourg; RCS B222.310; bank number LUB00000408; LEI 213800W1NGBLERUS6M39; BIC BCIRLULL; incorporated 6 Feb 2018. Authorised as a credit institution under Art. 2 of the Luxembourg Law of 5 April 1993 on the financial sector and CRR Art. 4(1)(1); home supervisor CSSF.  
   Sources: [regulatory information](https://www.bankingcircle.com/regulatory-information/). **Do not** take licence proof from the CSSF fraud warning about [“Circle Group S.A.”](https://www.cssf.lu/en/2025/01/warning-concerning-the-fraudulent-activities-carried-out-by-circle-group-s-a/) (possible name collision).

2. **Group map (official regulatory page + disclaimers, not the CASP press “About”).** EU/EEA branches of BCSA: Denmark (BIC SXPYDKKK), Germany (SXPYDEHH), Sweden (BCIRSE22), Norway (NUF, inc. 1 Mar 2025). UK: Third Country Branch as of 30 Nov 2023, PRA-authorised / FCA+PRA-regulated, FRN 848617, BIC SAPYGB2L. Wholly owned subsidiaries: Banking Circle (Liechtenstein) AG (FMA 288771, BIC BCIRLI22 / BCIRLI22XXX, CHF local clearing); BC Payments Pte. Ltd. (MAS Major Payment Institution, UEN 202201431Z, licence PS20200718 — Account Issuance, Domestic Money Transfer, Cross-border Money Transfer, E-money Issuance); Australian Settlements Limited trading as Banking Circle (APRA ADI, ABN 14 087 822 491, wholesale clients only; AU obligations not guaranteed by BCSA). Disclaimers also describe **BCUS, Inc.** (Banking Circle US) as a Connecticut commercial **uninsured** bank and a correspondent/sister, not a BCSA subsidiary.  
   Sources: [regulatory information](https://www.bankingcircle.com/regulatory-information/) · [disclaimers](https://www.bankingcircle.com/disclaimers/) · [MAS FID](https://eservices.mas.gov.sg/fid/institution/detail/433769-BC-PAYMENTS-PTE-LTD) · [MAS MPI press](https://www.bankingcircle.com/mas-grants-major-payment-institution-license-bc-payments/)

3. **B2B infrastructure bank: no retail / no direct-to-consumer accounts.** Not a retail bank, no personal accounts, no individuals. Risk appetite: banks, NBFIs, mid-to-large corporates. Physical cash pay-in/pay-out is prohibited.  
   Sources: [who is Banking Circle](https://www.bankingcircle.com/insights/who-is-banking-circle-understanding-our-role-in-payments/) · [risk appetite](https://www.bankingcircle.com/risk-appetite-policy/) · [banking services](https://www.bankingcircle.com/banking-services/)

4. **Integrator product set is accounts + local/cross-border payments + FX + treasury, white-label to FIs/PSPs/funds.** Physical and virtual multi-jurisdiction accounts (preferred IBAN/account prefix); COBO/POBO; bulk vIBAN; safeguarded/segregated client-money for regulated clients; payments via API / Client Portal / SFTP / SWIFT; FX on the same stack; FlexLiquidity physical cash pooling; fiduciary deposits (EUR, GBP, USD “and more to follow”) in an AA-rated bankruptcy-remote structure. Homepage audience: Payment businesses, Banks, Investment Funds.  
   Sources: [accounts](https://www.bankingcircle.com/accounts/) · [payments](https://www.bankingcircle.com/payments/) · [FX](https://www.bankingcircle.com/foreign-exchange/) · [FlexLiquidity](https://www.bankingcircle.com/flexliquidity/) · [deposits](https://www.bankingcircle.com/deposits/) · [home](https://www.bankingcircle.com/)

5. **Payment rails (cut-off table vs marketing).** Same-day/instant table (CET/CEST) documents **24 currencies**. Instant 24/7: EUR SEPA Inst, GBP Faster Payments, DKK TIPS, SEK RIX-Inst, AUD NPP. RTGS/batch examples: EUR T2 + SEPA SCT; GBP CHAPS; DKK T2 + Intradag; SEK RIX + Data Clearing; CHF SIC (via LI entity); AUD BECS (via AU entity); SWIFT for the rest (AED, CAD, CNH, CZK, HKD, HUF, ILS, JPY, MXN, NOK, NZD, PLN, RON, SAR, SGD, TRY, USD, ZAR, etc.). Direct-clearing marketing additionally claims SEPA Direct Debit (Core) and lists GBP Faster Payments (not CHAPS) as the GBP direct rail. Incoming/internal processing stops 19:00 CET/CEST. Same-day execution is not guaranteed if compliance, lack of funds, or policy holds apply. FX: funds forwarded to correspondent only after the FX trade settles.  
   Sources: [currencies and cut-off times](https://www.bankingcircle.com/currencies-and-cut-off-times/) · [infrastructure](https://www.bankingcircle.com/modern-and-secure-bank-infrastructure/) · [payments](https://www.bankingcircle.com/payments/)

6. **Agency banking vs correspondent vs own-brand.** Agency: regulated entities in the **SEPA zone (EUR)** or **UK (EUR and GBP)** **with a SWIFT BIC** get indirect SEPA SCT/Instant and Faster Payments using **their own BIC and IBANs**. Correspondent: own BIC/account/clearing code as visible ordering entity; 24 currencies and “settle 11 currencies locally”; T2 participation; typically reciprocal Nostro/Vostro over SWIFT. Correspondent clients must be regulated AML/CTF entities in a closed country list (EEA, UK incl. Gibraltar excl. Crown dependencies, CH, US, CA, AU, NZ, HK, SG, JP, CN, IN, MY, KR, TH).  
   Sources: [agency banking](https://www.bankingcircle.com/agency-banking/) · [correspondent banking](https://www.bankingcircle.com/correspondent-banking/) · [SWIFT](https://www.bankingcircle.com/swift/) · [risk appetite](https://www.bankingcircle.com/risk-appetite-policy/)

7. **Collections/payouts: SDD + Request-to-Pay exist; Bacs is inbound-only; Banking Circle does not issue cards.** SDD (EUR, 36 SEPA countries) with Dynamic Mandate Management via API; RtP as Open Banking PIS in EUR (SCT/SCT Inst) and GBP (Faster Payments). Help centre: Bacs GBP, 3 working days, **inbound only**. `/issuers/` is **card-programme funding/reconciliation via vIBANs**, not a Banking Circle card scheme. Direct Debit Connect APIs sit on **separate Azure hosts**.  
   Sources: [DD + RtP PDF](https://www.bankingcircle.com/wp-content/uploads/2024/11/Banking-Circle-Direct-Debit-and-RtP.pdf) · [KA-01065](https://support.bankingcircle.com/article/KA-01065/en-us) · [issuers](https://www.bankingcircle.com/issuers/) · [create first DD collection](https://docs.bankingcircleconnect.com/docs/create-your-first-direct-debit-collection) (indexed; live fetch login-walled)

8. **Digital assets: EMT issuer (EURI) + claimed CASP (15 Apr 2026) + fiat↔stablecoin settlement.** EURI is Banking Circle S.A.’s MiCA e-money token (1:1 EUR, redeemable at par). Official CASP announcement: CSSF CASP licence received **15 April 2026**; settlement offering including USDC, USDG and EURI. A live CSSF CASP register row was **not** retrieved (CSSF CASP page 404’d). Risk appetite: a Banking Circle-issued stablecoin “may only be marketed to crypto-asset service providers already onboarded”; CASPs have extra KYT/travel-rule conditions. EMT-issuer “independent CSSF confirmation” in the source briefing pointed at the **Circle Group S.A. fraud warning** — treat that as **unverified** for Banking Circle S.A.  
   Sources: [EURI launch](https://www.bankingcircle.com/banking-circle-launches-the-first-bank-backed-mica-compliant-stablecoin-euri/) · [stablecoin settlement](https://www.bankingcircle.com/banking-circle-introduces-stablecoin-settlement-services/) · [eurite.com about](https://www.eurite.com/about-us/) · [risk appetite](https://www.bankingcircle.com/risk-appetite-policy/)

9. **Onboarding is not self-serve; eligibility is published for UK/EEA NBPSPs and is tight.** NBPSP (PI/EMI, including late-stage authorisation) must complete BCICQ; be authorised or “minded to approve”; meet minimum revenue **or** pay minimum fees; safeguarding letter if needed; pass ownership/SoW, MLRO independence, TM/sanctions, etc. **UK: micro-enterprises are refused** (<10 persons and turnover/balance sheet ≤ €2m). Documents “available upon request.” Country Risk Matrix referenced but not published. Reverse solicitation only outside licensed territories.  
   Sources: [NBPSP eligibility](https://www.bankingcircle.com/eligibility-criteria-for-nbpsps/) · [risk appetite](https://www.bankingcircle.com/risk-appetite-policy/)

10. **Connect API is M2M Bearer JWT, sandbox vs production split, certificates not self-service.** Official ISO guides: authenticate first, then `Authorization: Bearer <token>`; Connecting to the API + M2M user token are login-walled. Indexed/support: sandbox `sandbox.bankingcircleconnect.com`; production `bankingcircleconnect.com` / `www.bankingcircleconnect.com`; token `https://authorizationsandbox.bankingcircleconnect.com/` (sandbox) and `https://www.bankingcircleconnect.com/api/v1/authorizations/authorize` (prod). Client Portal is a separate PSD2 UI (SCA via Authy) at https://www.bankingcircleconnect.com/.  
    Sources: [pain.001 support guide](https://support.bankingcircle.com/_entity/annotation/df9aa16c-1f8a-ee11-8179-0022489b0bca) · [camt.053 support guide](https://support.bankingcircle.com/_entity/annotation/255a7b91-c58a-ee11-8179-002248998b4c) · [connecting to the API](https://docs.bankingcircleconnect.com/guides/connecting-to-the-api) · [M2M user token](https://docs.bankingcircleconnect.com/api-reference/m2m-user-token) · [client portal](https://www.bankingcircle.com/client-portal/)

11. **Confirmed API resource groups** — see [API hosts](#api-hosts). Sources: ISO pain.001 / camt.053 guides · [ISO 20022](https://docs.bankingcircleconnect.com/v2/docs/iso20022) · [account report](https://docs.bankingcircleconnect.com/reference/get_api-v1-reports-bank-to-customer-account-report-1) · [virtual accounts](https://docs.bankingcircleconnect.com/docs/virtual-accounts-reference-data) · [API sandbox](https://docs.bankingcircleconnect.com/guides/api-sandbox)

12. **BC-NOW is internal 24/7 settlement between BCSA accounts, not a public scheme.** Real-time transfers across company IDs / entities / other clients on the BCSA books; opt-in participant directory (incomplete). Separate from SEPA Inst / FPS.  
    Sources: [payments](https://www.bankingcircle.com/payments/) · [digital assets institutions](https://www.bankingcircle.com/digital-assets-institutions/)

13. **Several “live rail” claims are not yet, or not consistently, documented as direct participation.** Payments page footnotes SIC IP (CHF) as “Scheduled to go live in 2026.” June 2026 insight: Banking Circle is **“set to become”** a direct CHAPS participant after BoE RTGS renewal — while the cut-off table already lists GBP CHAPS (offered; **direct** vs **indirect** not proven on the regulatory page). Local-clearing counts conflict: “up to 14 local payment currencies” vs “settle 11 currencies locally” vs “clearing in seven currencies” vs instant list EUR/GBP/DKK/SEK/AUD/CHF. Apr 2026 CASP press “About” adds Poland and Czech Republic branches; **regulatory-information and 10 Apr 2026 disclaimers do not.**  
    Sources: [payments](https://www.bankingcircle.com/payments/) · [CHAPS insight](https://www.bankingcircle.com/insights/chaps-direct-participant-banking-circle/) · [correspondent](https://www.bankingcircle.com/correspondent-banking/) · [banks](https://www.bankingcircle.com/banks/) · [stablecoin settlement](https://www.bankingcircle.com/banking-circle-introduces-stablecoin-settlement-services/) · [regulatory information](https://www.bankingcircle.com/regulatory-information/)

## Open questions

Do not invent answers.

- Full Connect reference (auth handshake, mTLS cert profile, token TTL/refresh, rate limits, idempotency beyond pain.001, error catalogue) is behind password. Webhook crypto is already in [Setup webhooks](webhooks/setup-webhooks.md).
- How a prospect gets sandbox credentials (relationship-manager gated?).
- Live CSSF CASP register row and exact MiCA service permissions — not retrieved (CASP URL 404). Relying on the 15 Apr 2026 announcement only.
- FCA/PRA live register body for FRN 848617 — Cloudflare blocked. UK branch status is from BC’s own regulatory page.
- CSSF Search Entities credit-institution record not pulled.
- FMA Liechtenstein, APRA ADI list, Connecticut DOB live register pages not opened; licences from BC disclaimers + local sites.
- Exact list of 11 vs 14 “local” currencies; which rails are **direct** vs **partner/correspondent**; whether CHAPS is already **direct**.
- Poland / Czech Republic branches claimed in press, absent from regulatory/disclaimers.
- Minimum fees / revenue thresholds for NBPSPs — exist, numbers not published.
- Country Risk Matrix, full prohibited-country list, holiday calendars.
- vIBAN country prefixes actually live (LU/DK/GB/DE commonly marketed; “and more” not enumerated).
- Cards: no issuer BIN/scheme product documented.
- TARGET vs T2 naming; OCT Inst (EPC register claimed Jun 2026 — not independently opened).
- “USD/CHF direct in 2H 2024” from a Nov 2024 marketing PDF — not confirmed as direct on the 2026 infrastructure page (USD is SWIFT-only in the cut-off table; CHF SIC is via LI).
- ISO 20022 Implementation Guide not retrieved.
- SFTP specification, SWIFT RMA/BIC usage, Client Portal API overlap.
- Stablecoin settlement API (mint/burn, chains, cut-offs, which legal entity books the trade) — press only.
- Interest rates, FX margin grid, landing fees (PLN noted), credit/overdraft models for SDD.
- Whether Connect API covers US / BCUS.
- Holding company / ownership (2024 accounts: B Circle Holding S.A. / BC MidCo Pte. Ltd.) — not fully parsed.

## Stale vs marketing vs real docs

| Item | Verdict |
|---|---|
| [Currencies and cut-off times](https://www.bankingcircle.com/currencies-and-cut-off-times/) | **Operational.** Best public rail/cut-off source. |
| [Regulatory information](https://www.bankingcircle.com/regulatory-information/) · [disclaimers](https://www.bankingcircle.com/disclaimers/) | **Legal.** Prefer over press “About” boxes. |
| [NBPSP eligibility](https://www.bankingcircle.com/eligibility-criteria-for-nbpsps/) · [risk appetite](https://www.bankingcircle.com/risk-appetite-policy/) | **Operational policy.** Use for onboarding design. |
| ISO pain.001 + camt.053 support guides | **Real integrator docs** (2022/2023 dated). Point at login-walled Connect portal for current auth. |
| docs.bankingcircleconnect.com | **Real docs, login-walled.** Indexed snippets can be stale vs v2. |
| Direct Debit + RtP PDF (Nov 2024) | **Marketing PDF** with useful scheme facts; not an API spec. Azure DD hosts only in Connect docs. |
| Overall Offering PDF (Nov 2024) | **Stale marketing.** Prefer 2026 cut-off table. |
| Homepage “€1.5T / 900+ FIs / 35M+ VAs / 10% European B2C e-commerce*” | **Marketing.** Asterisk cites Worldpay 2022. CASP press uses “750” FIs and “€1 trillion” — inconsistent. |
| “White-labelled FX”, “architects of virtual accounts”, “99.999% SWIFT availability” | **Marketing.** No SLA PDF opened. |
| Jun 2026 CHAPS insight | **Forward-looking.** “Set to become” ≠ confirmed live direct membership. |
| SIC IP “scheduled 2026” | **Not live** as of the payments page text. |
| CASP/stablecoin settlement (Apr 2026) | **Official announcement**, not an API/product spec. CSSF CASP register not independently listed. |
| EURI + eurite.com | **Issuer site.** Some eurite.com figures look placeholder. Prefer white paper / CSSF-ESMA EMT register for legal terms. |
| Australia NPP/BECS/BPAY/PayID/PayTo | **AU site marketing.** BPAY/PayID/PayTo are **not** on the group cut-off table (NPP + BECS are). |
| Third-party GitHub SDKs | **Not primary.** |

URLs actually opened in the research pass are listed in the source briefing; the ones that returned full public content are the cut-off table, regulatory/disclaimers, eligibility, risk appetite, MAS FID, local entity sites, and the two ISO support guides.

*Compiled 2026-08-29. Re-check login-walled Connect docs and CSSF/FCA/MAS registers before build.*
