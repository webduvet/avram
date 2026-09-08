# B2B / B2B2C fintech landscape map (payments integrator view)

> Snapshot **2026-09-08**; educational only—not legal advice; primary sources spot-checked **2026-09-08** (see Verification notes).

Related: [Banking Circle](../banking-circle/index.md) · [B4B Payments](../b4b-payments/index.md) · [Study material](../../study-material/index.md) · [Fintech index](../index.md)


## Sources & methodology

Primary company sites, partner docs, and regulator pages (FCA, CSSF, Bank of Lithuania, BaFin, ECB, EPC, Federal Reserve). Prefer EU/UK-relevant players. Licence types and FRNs stated only where publicly positioned on vendor regulatory pages or registers as of the snapshot date—**no invented licence numbers or fee tables**. Educational map for engineers integrating payments (especially Banking Circle + B4B); **not legal advice**. Re-check the relevant register and vendor legal pages before contracting.

**Snapshot date:** 8 Sep 2026 (Europe/Dublin).

Optional internal study pack (engineering notes, not a primary source): https://github.com/webduvet/avram/tree/main/docs/study-material

---

## How to read this map

Think in **capabilities**, not logos. A payments integration almost never buys “a bank” or “a fintech”; it buys a small set of services—rails to move money, a regulated wrapper to hold e-money or issue cards, a way to collect from merchants, and tools for KYC, fraud, ledgering, and reconciliation. Companies often span several columns because they sell a **product suite** (accounts + FX + cards + API) under one brand, while neighbours sell a **single capability** (e.g. issuer processing only, or AML screening only). Banking Circle sits near the **wholesale bank / clearing** end; B4B sits near **EMI / prepaid / card programmes / partner licensing**, powered by Banking Circle for accounts, payments, and FX. Use the table for orientation, then the detailed sections for differentiation and primary URLs.

**Out of deep scope for this map:** bancassurance / specialty insurance products (omit); deep wealth/robo advisory (one-line adjacent only).

### Licence-type legend (integrator shorthand)

| Tag | Meaning (educational) | Client-money failure mode (high level) |
| --- | --- | --- |
| **CI** | Credit institution (bank) — deposit-taking | Deposits may benefit from deposit-guarantee schemes (e.g. UK **FSCS**, EU **DGS**) where eligible—not e-money |
| **EMI** | Electronic money institution — issues e-money | **Safeguarding** (segregation / insurance-guarantee methods); **not** FSCS on the EMI balance |
| **PI** | Payment institution — payment services; may or may not hold funds long-term | Safeguarding where relevant funds rules apply; AISP/PISP-only firms often do not hold float |
| **AISP / PISP** | Account-information / payment-initiation under PSD2 / UK OB | Consent-based access to ASPSP accounts; not a substitute for acquiring or for holding balances |
| **CASP** | Crypto-asset service provider (e.g. MiCA) | Distinct regime from EMI/CI; Travel Rule / AML overlays apply |

See **Safeguarding vs bank deposits** and the Banking Circle ↔ B4B table for the practical contrast.

---

## Category overview

| Category | What it is (short) | Example companies |
| --- | --- | --- |
| Credit institution / wholesale bank rails | Licensed bank providing B2B accounts, clearing, VIBANs, FX, correspondent access for PSPs/FIs | Banking Circle, ClearBank, JPMorgan Payments, Citi TTS |
| EMI / e-money / prepaid platforms | E-money accounts, prepaid balances, safeguarding; often the regulated front for programmes | B4B Payments, Modulr, OpenPayd, Treezor, Swan, Equals |
| Payment institution / AISP / PISP / open banking | PSD2/UK OB account info & payment initiation (pay-by-bank); some also hold EMI | TrueLayer, Tink, Plaid, Token.io, Yapily |
| Acquiring / merchant acquiring / gateway | Accept cards (and often A2A) from buyers; settle to merchant | Stripe, Adyen, Worldpay (Global Payments), Checkout.com, Mollie |
| Card issuing / BIN sponsorship | Programme + BINs so you can issue branded cards without full scheme membership | B4B, Marqeta, Thredd (+ BIN sponsors), Galileo, Paymentology |
| Card scheme / network | Rules, brand, clearing between issuers and acquirers | Visa, Mastercard (also UnionPay, Amex — brief) |
| Banking-as-a-Service / embedded finance | White-label accounts/cards/payments under partner bank or EMI licence | Solaris, ClearBank (embedded), Equals, Unit, Treasury Prime, Cross River partnerships |
| FX / multi-currency / treasury payments | Convert currencies; pay out to bank accounts globally | Wise Platform, Currencycloud, Banking Circle FX, Airwallex |
| Payouts / disbursements | Mass seller/contractor/marketplace payouts to cards or accounts | Tipalti, Payoneer, MassPay, Hyperwallet (PayPal), B4B payout cards |
| Settlement / reconciliation / banking status APIs | Confirm funds moved; match statements; webhooks/ISO status | Banking Circle Connect, Modern Treasury, ClearBank APIs, Atlar |
| KYC / KYB / identity / AML screening | Onboard people/companies; sanctions/PEP/adverse media | Onfido, Sumsub, ComplyAdvantage, Persona, Jumio |
| Fraud / transaction monitoring | Real-time abuse, ATO, payment fraud, ongoing AML monitoring | Feedzai, Unit21, Sardine, Hawk, Flagright |
| Ledger / virtual accounts / treasury mgmt | Internal books, VIBANs/sub-accounts, cash visibility | Banking Circle VIBANs, Formance, Modern Treasury Ledgers, Atlar |
| Stablecoin / crypto on-ramps (brief) | Fiat↔stablecoin for B2B settlement/treasury | Circle Mint, Banking Circle stablecoin settlement (CASP), Fireblocks |
| Correspondent banking / SWIFT / clearing (brief) | Cross-border messaging & euro/UK/US clearing plumbing | SWIFT, T2/TARGET, EBA Clearing STEP2/RT1, Fedwire/FedACH/FedNow |
| **Control & scheme layer — SCA / 3DS / tokenization** | Authenticate CNP; network tokens vs PAN | EMVCo 3DS, Cardinal, Visa Token Service, Mastercard MDES |
| **Control & scheme layer — chargebacks / disputes** | Representment, pre-dispute alerts, scheme monitoring | Visa/MC rules, Verifi RDR, Ethoca, Stripe/Adyen dispute tools |
| **A2A rails beyond open banking** | Instant credit / RTP schemes (not TPPs) | SCT Inst, TIPS, RT1, Faster Payments, FedNow, RTP, Request-to-Pay |
| Core banking / processor platforms (brief) | Cores and issuer/acquiring hosts under BaaS stacks | Thought Machine, Mambu, Temenos, Thredd, Marqeta |

### Often-missed integrator layers

| Layer | Why it bites late | Jump to |
| --- | --- | --- |
| SCA / 3DS / network tokens | EU/UK CNP declines, liability shift, vault design | Control & scheme — SCA / 3DS / tokenization |
| Chargebacks / RDR–Ethoca-class alerts | Margin, scheme monitoring programmes, ops SLAs | Control & scheme — chargebacks / disputes |
| Interchange & MDR economics | Card vs A2A unit economics (no fake fee tables) | Interchange & economics |
| Safeguarding vs FSCS/DGS | EMI insolvency ≠ bank deposit insurance | Safeguarding vs bank deposits |
| A2A rails vs TPP aggregators | SCT Inst / FPS / FedNow ≠ TrueLayer/Plaid | A2A rails beyond open banking |
| Travel Rule / sanctions (CASP) | Stablecoin/CASP corridors (e.g. BC CASP) | RegTech beyond KYC |
| Settlement/recon vs ledger/VIBAN | Status APIs ≠ programmable books ≠ routing refs | Categories 10 and 13 |

---

## Why one company appears in many columns

**Product suite vs single capability.** Suites (Stripe, Adyen, B4B, Banking Circle Group offerings, Wise Platform, Equals) bundle adjacent jobs—accept, hold, convert, pay out, sometimes issue cards—so they correctly appear in multiple categories. Specialists (Thredd = issuer processing; ComplyAdvantage = screening; Formance = ledger) own one layer and expect you to compose the rest.

**How an integrator typically stitches 2–4 vendors**

1. **Collect** — acquirer/gateway (and/or open-banking PISP) takes money from the buyer.  
2. **Hold & regulate** — EMI or credit institution holds client money (safeguarding or bank deposits), often with **virtual accounts/VIBANs** for routing.  
3. **Move & convert** — bank rails / FX / payouts send funds to sellers, cards, or treasury.  
4. **Control plane** — KYC/KYB + fraud/TM + your **ledger** + reconciliation against bank/PSP webhooks or ISO 20022 status + (for cards) 3DS/tokens and dispute ops.

Example stitch: **Adyen/Stripe → B4B (EMI + programme) → Banking Circle (rails/VIBAN/FX) → Modern Treasury or Atlar (recon)** — with Sumsub + Feedzai/Flagright on the side. Graduating partners may drop the EMI wrapper and go **direct to Banking Circle** once licensed and at scale (B4B’s published “Banking Circle Direct” path).

---

## Safeguarding vs bank deposits

**What it is**  
Two different client-money protection models that look similar in a product UI (“balance”) but diverge on insolvency:

- **EMI / PI relevant funds** — must be **safeguarded** (segregated account at a credit institution, or insurance/comparable guarantee). Aim: return value if the payments firm fails. Funds on an EMI are **not** protected by UK **FSCS** deposit insurance (FSCS protects eligible deposits at PRA-authorised banks/building societies/credit unions).  
- **Credit-institution deposits** — money held as a bank deposit may be covered by deposit-guarantee arrangements (UK FSCS; EU Deposit Guarantee Schemes) within eligibility limits and firm type.

**How it differs (Banking Circle vs B4B)**  
Banking Circle is a **CSSF credit institution** (with a UK third-country branch)—wholesale **bank** rails/accounts. B4B is an **EMI** front (UK FCA + LT BoL) that safeguards e-money and often sits on Banking Circle infrastructure (“Powered by Banking Circle”). Integrators should not assume “app balance = FSCS-protected deposit.”

**Primary URLs**  
- https://www.fca.org.uk/consumers/using-payment-service-providers  
- https://www.fscs.org.uk/news/protection/e-money-and-fscs-protection/  
- https://www.fca.org.uk/firms/emi-payment-institutions-safeguarding-requirements  
- https://www.fca.org.uk/publications/policy-statements/ps25-12-changes-safeguarding-regime-payments-and-e-money-firms  
- https://www.bankingcircle.com/regulatory-information/  
- https://www.b4bpayments.com/c/regulatory  

---

## Detailed categories

### 1. Credit institution / wholesale bank rails

**What the buyer gets**  
A regulated **bank** relationship aimed at other financial institutions and payments businesses: multi-currency accounts, payment initiation into local clearing (e.g. SEPA, Faster Payments), FX, virtual IBANs, and often correspondent/agency-style connectivity—not a retail current account for consumers.

**How it differs**
- Deposit-taking **credit institution** (not EMI/PI); typically deeper clearing access and bank-grade deposit regimes where applicable.
- Neighbours: EMIs issue e-money; acquirers collect card payments; BaaS platforms may *use* a bank underneath.
- Buyers are usually PSPs, banks, funds—not end consumers.

**Regulatory angle (sourced)**  
Banking Circle S.A.: as of 2026-09-08 publicly positions as a **credit institution** under Luxembourg law, supervised by the **CSSF** (financial registration **LUB00000408**); UK third-country branch PRA/FCA (FS Register **848617**). ClearBank publicly positions as a **fully regulated bank** in the UK and Europe (BoE / DNB–ECB fund holding described on site; ClearBank Limited FRN **754568** commonly cited in ClearBank materials—re-check register before contract).

**Representative companies**  
- **Banking Circle** — B2B payments bank; VIBANs, FX, clearing, Connect API; ~24 fiat + stablecoins per homepage; sister group with B4B.  
- **ClearBank** — UK cloud-native clearing bank; agency/embedded/transaction banking APIs.  
- **JPMorgan Payments / Citi TTS** — large-bank wholesale payments/treasury (global).  
- **Deutsche Bank / Barclays / HSBC** corporate & institutional rails (incumbents).  
- **Cross River** — US sponsor bank often paired with fintech programmes (US leg for some B4B products).

**Primary URLs**  
- https://www.bankingcircle.com  
- https://www.bankingcircle.com/regulatory-information/  
- https://clear.bank/  

---

### 2. EMI / e-money / prepaid / card issuing platforms

**What the buyer gets**  
Regulated **e-money** accounts and prepaid balances (and often cards/spend tools): the legal wrapper that issues e-money, safeguards client funds, and runs programmes for businesses or embedded partners—without being a full deposit bank.

**How it differs**
- EMI ≠ credit institution: typically **cannot** take deposits/lend like a bank; funds are safeguarded e-money.  
- Overlaps with card issuing and BaaS; differs from pure PI open-banking players who may not hold balances the same way.  
- B4B explicitly positions as EMI front + cards/accounts, **powered by Banking Circle** for bank infrastructure.

**Regulatory angle (sourced)**  
B4B: **Payment Card Solutions (UK) Limited** — as of 2026-09-08 publicly positions as FCA EMI under Electronic Money Regulations 2011 (**Ref 930619**); **UAB B4B Payments Europe** — Bank of Lithuania EMI (**Licence No. 76**); US Visa prepaid via **Cross River Bank, Member FDIC**. Modulr / OpenPayd commonly publicly position as EMIs (re-check registers before contract).

**Representative companies**  
- **B4B Payments** — accounts, payments, FX, prepaid/expense/payout cards; partner models including BIN sponsorship and “operate under B4B licences”; Banking Circle Group sister.  
- **Modulr** — EMI payments automation, virtual accounts, UK scheme connectivity.  
- **OpenPayd** — EMI multi-currency / vIBAN infrastructure.  
- **Treezor, Swan** — EU embedded EMI-style platforms.  
- **Equals (formerly Railsr / Equals Money)** — as of 2026-09-08 publicly positions as UK FCA-regulated EMI/PI; embedded payments, accounts, cards, FX; Railsr BaaS/CaaS heritage rebranded **Equals** (company news **1 Jun 2026**).  
- **Prepaid Financial Services Limited (PFS)** — UK EMI under **EML** group; as of 2026-09-08 T&Cs state FCA e-money issuer **FRN 900036** (confirm programme availability before treating as a peer to B4B/Marqeta).  
- **UAB Finansinės paslaugos Contis** — LT EMI in the **Paysera** group (BoL-approved acquisition; Paysera has stated it is the sole remaining Contis client)—**not** the same entity as PFS UK; do not lump as “Contis-class.”

**Primary URLs**  
- https://www.b4bpayments.com  
- https://www.b4bpayments.com/c/regulatory  
- https://www.b4bpayments.com/c/solutions/partners  
- https://equalsmoney.com/newsroom/equals-money-railsr-becomes-equals  
- https://prepaidfinancialservices.com/terms-conditions  
- https://www.paysera.com/v2/en/blog/paysera-acquisition-of-contis-approved  

*Note on **Oversight** (integrator API):* Partner documentation at https://b4bpayments.readme.io (password-gated) documents **Oversight** as a separate API surface (`sandbox.b4bpayments.com/oversight/v1`) for **regulatory / pre-settlement** payment intake: screening, TM, then handoff to Banking Circle. Published Oversight contract: terminal statuses `B4BTMApproved` / `B4BFailed`; settlement tracking is Banking Circle’s job via Connect webhooks/GETs. Public marketing pages emphasise EMI/partner licensing and “Powered by Banking Circle”; the Oversight product name lives mainly in the partner ReadMe, not the open homepage.

---

### 3. Payment institution / AISP / PISP / open banking

**What the buyer gets**  
APIs to **read account data (AISP)** and/or **initiate payments from a user’s bank account (PISP)** under PSD2 / UK Open Banking—with user consent—enabling pay-by-bank, affordability checks, and account linking without card schemes.

**How it differs**
- Does not replace acquiring for cards; complements or competes on A2A checkout.  
- Usually does **not** hold the merchant’s float long-term the way an EMI/bank does—**unless** the same firm also holds EMI (or similar) permissions.  
- Distinct from **A2A rails** (SCT Inst, FPS, FedNow): TPPs sit on top of ASPSP APIs / scheme access; rails are the clearing/settlement systems themselves (see **A2A rails beyond open banking**).

**Regulatory angle (sourced)**  
TrueLayer: as of 2026-09-08 publicly positions as **Authorised Payment Institution** with AIS and PIS (FCA **901096**), and states it **also acquired an EMI licence** under the same FRN allowing it to hold and settle funds (TrueLayer security page)—re-check register permissions before contract. Plaid Financial Ltd: FCA authorised under Payment Services Regulations (FRN **804718**). Equivalent NCA authorisations apply in EEA.

**Representative companies**  
- **TrueLayer** — EU/UK open banking payments & data; EMI permission under FRN 901096 per security page; Stripe UK Pay by Bank partner (announced).  
- **Tink** (Visa) — open banking / account aggregation.  
- **Plaid** — data + payments connectivity; agent models for AIS.  
- **Token.io, Yapily, Powens** — connectivity / TPP infrastructure.  
- Bank-native OB APIs (ASPSPs) — the other side of the pipe.

**Primary URLs**  
- https://truelayer.com/  
- https://truelayer.com/security/security-at-truelayer/  
- https://plaid.com/open-banking/  
- https://tink.com/  

---

### 4. Acquiring / merchant acquiring / payment gateway

**What the buyer gets**  
Ability for a **merchant** to accept card (and often local/A2A) payments online or in-person: gateway + acquiring, fraud tools, settlement to the merchant’s bank/EMI account, reporting and chargeback handling.

**How it differs**
- Faces the **buyer→merchant** collection problem; payouts/disbursements face **platform→seller**.  
- May bundle issuing, banking, or pay-by-bank via partners (e.g. Stripe + TrueLayer).  
- Not a substitute for wholesale correspondent rails (though large PSPs use those behind the scenes).

**Regulatory angle**  
Varies widely: many EU entities are PIs/EMIs and/or work with acquiring banks. **Adyen N.V.**, as of 2026-09-08, publicly positions as authorised as a **credit institution** under **De Nederlandsche Bank (DNB)** (“banking license”), including cross-border acquiring/payment/banking services in the EEA under CRD IV passporting ([Adyen licences — EMEA](https://www.adyen.com/licenses/emea)); DNB public register lists Category Bank - CI, relation **F0001**, activities from **25 Apr 2017**. Homepage also states “Backed by US, UK, and EU banking licenses.” Re-check live DNB/local entities before contract. Do not assume “Stripe = bank” in the EU.

**Representative companies**  
- **Stripe, Adyen, Worldpay (Global Payments), Checkout.com, Mollie, Braintree** — Global Payments **completed** the acquisition of Worldpay from FIS and GTCR (and divestiture of Issuer Solutions to FIS) on **9 Jan 2026** ([Form 8-K](https://investors.globalpayments.com/financial-information/all-sec-filings/content/0001104659-26-002705/tm262856d1_8k.htm)); the completion PR was **released 12 Jan 2026** ([GPN investors PR](https://investors.globalpayments.com/news-events/press-releases/detail/498/global-payments-completes-acquisition-of-worldpay-and)).  
- Regional: **Nexi, Worldline, Elavon**  
- Best known for something else but play here: **PayPal** (wallet + acquiring-like), **Amazon Pay**

**Primary URLs**  
- https://stripe.com  
- https://www.adyen.com  
- https://help.adyen.com/en_US/knowledge/finance/balances/how-am-i-assured-that-my-funds-are-safe-at-adyen-nv  
- https://www.dnb.nl/en/public-register/information-detail/?registerCode=WFTDG&relationNumber=F0001  
- https://checkout.com  
- https://investors.globalpayments.com/news-events/press-releases/detail/498/global-payments-completes-acquisition-of-worldpay-and  

---

### 5. Card issuing / BIN sponsorship

**What the buyer gets**  
Infrastructure to **create and manage payment cards** (virtual/physical, prepaid/debit): PAN/token lifecycle, controls, wallets—often via a **BIN sponsor** (scheme member) if you lack principal membership.

**How it differs**
- Issuing ≠ acquiring; schemes connect both.  
- **Issuer processor** (Thredd, Marqeta) may be separate from **BIN sponsor / EMI** (B4B).  
- B4B partner page: BIN sponsorship, settlement-only, full scheme support, and graduation to Banking Circle Direct; processing via **Thredd** called out for white-label programmes.

**Regulatory angle**  
BIN sponsor / EMI or bank must hold appropriate e-money/credit-institution permissions and scheme membership. B4B: Mastercard principal (UK/EU) and Visa partner via Cross River (US) per public site.

**Representative companies**  
- **B4B Payments** — BIN sponsorship + prepaid/expense/payout cards.  
- **Marqeta, Thredd, Galileo, Paymentology, i2c** — issuing platforms/processors.  
- **Wallester, NymCard** — regional/API issuers.  
- US BaaS banks (**Unit, Cross River**) often bundle debit issuing.

**Primary URLs**  
- https://www.b4bpayments.com/c/solutions/partners  
- https://www.thredd.ai/  
- https://www.marqeta.com  

---

### 6. Card scheme / network (brief)

**What the buyer gets**  
The **rulebook and network** that authorises, clears, and settles card transactions between issuers and acquirers, plus brand acceptance worldwide.

**How it differs**  
Schemes are not your bank or EMI; you access them through membership or a sponsor. Separate from A2A rails (SEPA, FPS) and from SWIFT messaging.

**Representative**  
**Visa, Mastercard**; also **American Express, Discover/Diners, UnionPay, JCB** in relevant corridors.

**Primary URLs**  
- https://www.visa.com  
- https://www.mastercard.com  
- https://developer.visa.com  
- https://developer.mastercard.com  

---

### 7. Banking-as-a-Service / embedded finance platforms

**What the buyer gets**  
APIs to embed **accounts, payments, and often cards/lending** into a non-bank product under a partner’s licence—white-label banking features without building a bank.

**How it differs**
- Orchestrates licence + ledger + cards + compliance UX; may be bank-led (Solaris, ClearBank) or EMI-led (Swan, Treezor, B4B embedded, Equals).  
- Overlaps with wholesale rails and EMI categories; buying BaaS is buying **the package + operating model**, not just a SEPA pipe.

**Regulatory angle (sourced)**  
Solaris publicly: German **CRR credit institution** (BaFin), EU passporting. ClearBank: regulated bank UK/EU. Unit / Treasury Prime: US partner-bank models. B4B: EMI + embedded partner programmes; US via Cross River. Equals: UK EMI/PI embedded platform (ex-Railsr).

**Representative companies**  
- **Solaris** — EU BaaS bank APIs (accounts, cards, etc.).  
- **ClearBank** — embedded/agency banking.  
- **B4B** — embedded cards/payouts/accounts for platforms.  
- **Equals (formerly Railsr / Equals Money)** — embedded payments/accounts/cards/FX (rebrand 1 Jun 2026).  
- **Unit, Treasury Prime, Cross River** — US embedded/sponsor-bank ecosystems.

**Primary URLs**  
- https://www.solarisgroup.com/en/license/  
- https://clear.bank/  
- https://www.b4bpayments.com  
- https://equalsmoney.com/newsroom/equals-money-railsr-becomes-equals  

---

### 8. FX / multi-currency / treasury / payments to bank accounts

**What the buyer gets**  
Convert currencies and **pay or collect** via local account details / bank transfers across corridors—often with multi-currency wallets or named accounts—aimed at platforms, FIs, and corporates.

**How it differs**
- Focus is **FX + A2A payout/collection**, not card acquiring.  
- Banking Circle offers FX and multi-currency as a **bank**; Wise Platform / Currencycloud are widely used **embedded FX/payments** layers. Currencycloud: **Visa-owned** (Visa completed acquisition **20 Dec 2021**—cite Visa PR, not homepage hero alone); licensed in UK, NL, and other markets per Currencycloud positioning.  
- Treasury tools (Atlar) may sit above multiple providers.

**Representative companies**  
- **Wise Platform, Currencycloud, Airwallex, Banking Circle, Ebury, OFX**  
- Bank TTS desks (JPM, Citi) for large corporates.

**Primary URLs**  
- https://wise.com/platform  
- https://www.currencycloud.com/  
- https://usa.visa.com/about-visa/newsroom/press-releases.releaseId.18666.html  
- https://www.bankingcircle.com  

---

### 9. Payouts / disbursements / marketplace seller payouts

**What the buyer gets**  
Orchestrated **mass pay-outs** to sellers, drivers, affiliates, or claimants—bank account, wallet, or **payout card**—plus payee onboarding, tax forms (where relevant), and status tracking.

**How it differs**
- Downstream of acquiring/marketplace escrow; optimises **last-mile disbursement**.  
- Can be pure orchestration (Tipalti) or card-based instant access (B4B payout cards) or global wallet (Payoneer).

**Representative companies**  
- **Tipalti, Payoneer, MassPay, Hyperwallet, Tremendous**  
- **B4B** — payout/incentive/expense cards  
- Acquirers’ payout products (Adyen for Platforms, Stripe Connect) — suites that also play here

**Primary URLs**  
- https://tipalti.com  
- https://www.payoneer.com  
- https://www.b4bpayments.com  

---

### 10. Settlement / reconciliation / banking APIs for status

**What the buyer gets**  
APIs/webhooks/files that answer: *Did the payment settle? To which virtual account? What’s bank/PSP reported status vs your expected amount?* Includes payment status reports (e.g. pain.002-style / ISO 20022 status flows), statement ingestion, and matching engines.

**How it differs from category 13 (ledger / VIBAN)**  
- **Cat 10 = visibility & matching over money movement** you already initiated on external rails (Connect webhooks, camt/pacs status, acquirer settlement reports).  
- **Cat 13 = your internal books + routing references** (programmable ledger balances you owe users; VIBANs as aliases on a licensed account).  
You typically need both: ledger truth internally, settlement/recon to prove the bank/PSP agrees.

**Representative companies**  
- **Banking Circle Connect**, **ClearBank APIs**, **Modulr webhooks**  
- **Modern Treasury**, **Atlar**, ERP bank connectors  
- Acquirer reconciliation (Adyen/Stripe reporting)  
- Card settlement education: Lithic transaction-flow (DMS vs SMS) + settlement blog

**Primary URLs**  
- https://docs.bankingcircleconnect.com/ (access may require credentials)  
- https://www.moderntreasury.com  
- https://www.atlar.com  
- https://docs.lithic.com/docs/transaction-flow  
- https://www.lithic.com/blog/what-is-settlement-processing  
- https://www.iso20022.org/catalogue-messages  
- https://www.swift.com/sites/default/files/files/swift-iso20022fordummies-6thedition-2022.pdf  

---

### 11. KYC / KYB / identity / AML screening vendors

**What the buyer gets**  
Vendor APIs/SDKs to **verify customers and businesses**, screen sanctions/PEP/adverse media, and support ongoing due diligence—usually complementary to your own licence obligations (you remain accountable).

**How it differs**  
Identity proofing (docs/biometrics) ≠ ongoing transaction monitoring ≠ bank account verification (open banking). Often 2–3 vendors in one stack. See also **RegTech beyond KYC** for Travel Rule / continuous sanctions ops.

**Representative companies**  
- **Onfido, Sumsub, Jumio, Persona** — identity / KYC-KYB  
- **ComplyAdvantage, Dow Jones, Refinitiv World-Check** — screening data  
- Suite players adding monitoring (Sumsub, etc.)

**Primary URLs**  
- https://onfido.com  
- https://sumsub.com  
- https://complyadvantage.com  

---

### 12. Fraud / transaction monitoring

**What the buyer gets**  
Real-time and batch systems to score **payment fraud, account takeover, mule activity**, and/or **AML transaction monitoring** with case management and SAR-oriented workflows.

**How it differs**  
Fraud decisioning at auth/payout time vs AML pattern detection over time; many platforms now blend both. Distinct from scheme chargeback tools and from KYC onboarding.

**Representative companies**  
- **Feedzai, FeatureSpace, Sardine, Hawk, Unit21, Flagright**  
- Acquirer-native fraud (Stripe Radar, Adyen RevenueProtect)  
- B4B publicly noted Flagright for AML/fraud capabilities (vendor PR / Flagright post)

**Primary URLs**  
- https://www.feedzai.com  
- https://www.unit21.ai  
- https://www.flagright.com  

---

### 13. Ledger / virtual accounts / treasury management

**What the buyer gets**  
(1) **Programmable ledger** of balances you owe users; (2) **virtual accounts/VIBANs** as routing references on a real bank/EMI account; (3) **treasury** cash visibility and payment control across entities.

**How it differs from category 10 (settlement / recon)**  
- Ledger/VIBAN is **state you own** (books + aliases). Settlement/recon is **evidence from the rail/PSP** that funds moved and how to match them.  
- Ledger can be software-only (Formance, Modern Treasury Ledgers) while VIBANs require a **licensed account provider** (Banking Circle, ClearBank, OpenPayd…).  
- Treasury platforms (Atlar) aggregate many banks/PSPs and often also do recon (overlap with cat 10).

**Representative companies**  
- **Banking Circle VIBANs**, **ClearBank virtual accounts**, **OpenPayd / Modulr**  
- **Formance, Modern Treasury Ledgers**  
- **Atlar** — treasury connectivity & recon  

**Primary URLs**  
- https://www.bankingcircle.com  
- https://www.formance.com  
- https://www.moderntreasury.com/products/ledgers  

---

### 14. Stablecoin / crypto on-ramps (brief, B2B-relevant)

**What the buyer gets**  
Institutional **fiat↔stablecoin** conversion and settlement for treasury or cross-border value transfer, sometimes integrated into bank platforms.

**How it differs**  
Adjacent to FX/rails; subject to **MiCA/CASP** and local rules—not a drop-in for SEPA. Banking Circle has publicly announced **stablecoin settlement** following **CSSF CASP** approval (per Banking Circle news, CASP cited **15 April 2026**). Circle Mint provides institutional USDC/EURC on-ramps with SEPA/SWIFT rails per Circle docs. Travel Rule / crypto-compliance tooling becomes relevant (see RegTech).

**Representative**  
**Circle (Mint/USDC/EURC), Banking Circle stablecoin services, Fireblocks, Anchorage** (custody—confirm fit per programme).

**Primary URLs**  
- https://www.circle.com  
- https://www.bankingcircle.com/banking-circle-introduces-stablecoin-settlement-services/  

---

### 15. Correspondent banking / SWIFT / clearing & settlement infrastructure (brief)

**What the buyer gets**  
The **plumbing**: cross-border messaging (SWIFT), RTGS and ACH-like systems (T2/TARGET, Fedwire), and CSM operators (EBA Clearing STEP2 for SEPA bulk; RT1 for instant). Banks/EMIs participate; most fintechs consume via a bank partner.

**How it differs**  
Infrastructure utilities vs commercial “products.” Banking Circle positions **direct clearing** (SEPA, SWIFT) and **correspondent banking** for clients (own BIC/accounts; API or SWIFT). Instant retail rails are detailed under **A2A rails beyond open banking**.

**Representative**  
**SWIFT; ECB T2 / TARGET Services; EBA Clearing (STEP2, RT1); Bank of England RTGS / Pay.UK Faster Payments; Fedwire / FedACH / FedNow / CHIPS (US).**

**Primary URLs**  
- https://www.swift.com  
- https://www.ebaclearing.eu/  
- https://www.ecb.europa.eu/paym/target/html/index.en.html  
- https://www.ecb.europa.eu/paym/target/t2/html/index.en.html  
- https://www.federalreserve.gov/paymentsystems/fedfunds_about.htm  
- https://www.frbservices.org/financial-services/ach  

---

## Control & scheme layer

### SCA / 3-D Secure / tokenization

**What it is**  
- **PSD2 SCA** — Strong Customer Authentication for remote electronic payments in the EEA/UK (challenge or risk-based exemptions such as TRA, low-value, MIT, trusted beneficiaries—details in RTS/guidance).  
- **EMV® 3-D Secure (3DS2)** — protocol exchanging device/transaction data between merchant and issuer ACS to authenticate CNP; often the SCA mechanism for cards.  
- **Tokenization** — replace PAN with a surrogate. **Network tokens** (Visa Token Service, Mastercard MDES) are issued/managed by the scheme token service; distinct from merchant/processor vault PANs or PCI tokens.

**How it differs**  
SCA is the regulatory obligation; 3DS is a common card implementation; network tokens reduce PAN exposure and can improve auth rates—but do not replace disputes ops or AML. Issuer processors (Thredd, Marqeta, Lithic-class) and 3DS servers (e.g. Cardinal) sit in the path.

**Example companies / systems**  
EMVCo (spec), Cardinal Commerce (Visa subsidiary; 3DS server), Visa Token Service, Mastercard MDES, issuer ACS / processors (Thredd, Marqeta).

**Primary URLs**  
- https://www.emvco.com/emv-technologies/3-d-secure/  
- https://usa.visa.com/products/visa-token-service.html  
- https://developer.visa.com/capabilities/token-service-provisioning/overview  
- https://developer.mastercard.com/mdes-digital-enablement/documentation/  
- https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32015L2366  
- https://www.eba.europa.eu  

---

### Chargebacks / disputes / scheme compliance ops

**What it is**  
Lifecycle after a card payment is challenged: retrieval/request for information → chargeback → **representment** (merchant/acquirer evidence) → pre-arbitration/arbitration under scheme rules. **Pre-dispute collaboration** tools (Visa **Verifi** including **RDR**; Mastercard **Ethoca** alerts) aim to refund or resolve before a formal chargeback. Acquirers expose dispute APIs/dashboards (Stripe, Adyen). Scheme monitoring programmes (dispute/fraud ratios) drive operational urgency.

**How it differs**  
Distinct from fraud decisioning at auth time and from A2A recalls/APP-fraud regimes. Issuer vs acquirer roles reverse the money flow; SLAs are scheme- and product-specific—do not invent fee tables.

**Example companies / systems**  
Visa Dispute / Rules materials; Mastercard chargeback/rules portals; Verifi RDR; Ethoca Alerts; Stripe Disputes; Adyen dispute docs.

**Primary URLs**  
- https://www.verifi.com/rdr-automatically-stop-chargebacks.html  
- https://www.ethoca.com  
- https://docs.stripe.com/disputes  
- https://docs.stripe.com/disputes/get-started/prevention  
- https://docs.adyen.com  
- https://www.visa.com  
- https://www.mastercard.com  

---

### A2A rails beyond open banking

**What it is**  
Account-to-account **clearing/settlement schemes** that move value between payment accounts—often instant—independent of card schemes. Open-banking **TPPs** (cat 3) initiate or read; **rails** settle.

| Rail / overlay | Region | Role (educational) |
| --- | --- | --- |
| **SCT / SCT Inst** | SEPA | EPC schemes for euro credit transfer / instant credit transfer |
| **TIPS** | Euro (Eurosystem) | TARGET Instant Payment Settlement in central bank money 24/7 |
| **EBA Clearing RT1 / STEP2** | SEPA | CSM for instant / bulk SEPA |
| **Faster Payments** | UK | Pay.UK near-instant GBP retail payments |
| **FedNow** | US | Federal Reserve instant payments service |
| **RTP® network** | US | The Clearing House real-time payments |
| **Request-to-Pay** | SEPA (SRTP); UK Pay.UK RtP | Messaging overlay to request a payment—not itself a settlement rail |
| **Fedwire / FedACH** | US | RTGS wires / batch ACH (not “instant retail” but core A2A plumbing) |

**How it differs**  
TrueLayer/Plaid/Tink ≠ SCT Inst/TIPS/FedNow. Integrators must know which rail their bank/EMI actually clears on.

**Primary URLs**  
- https://www.europeanpaymentscouncil.eu/what-we-do/epc-payment-schemes/sepa-instant-credit-transfer/sepa-instant-credit-transfer-rulebook  
- https://www.europeanpaymentscouncil.eu/what-we-do/other-schemes/sepa-request-pay  
- https://www.ecb.europa.eu/paym/target/tips/html/index.en.html  
- https://www.ebaclearing.eu/  
- https://www.wearepay.uk/  
- https://www.federalreserve.gov/paymentsystems/fednow_about.htm  
- https://www.frbservices.org/financial-services/fednow  
- https://www.theclearinghouse.org/payment-systems/rtp  

---

### Interchange & economics (brief)

**What it is**  
For **cards**, the merchant’s all-in acceptance cost (often called MDR) typically bundles: **interchange** (paid toward the issuer), **scheme/assessment fees**, and **acquirer/processor markup**. Interchange schedules are set by schemes (and constrained by regulation in some regions)—merchants negotiate markup/pricing model more than wholesale interchange. For **A2A / pay-by-bank**, economics usually follow PSP + rail tariffs without classic four-party interchange (still not “free”—scheme, bank, and TPP fees apply).

**How it differs**  
Card vs A2A unit economics drive product design (payout cards vs SEPA Instant). **EU Interchange Fee Regulation (IFR)** caps certain consumer card interchange in the EEA—read the regulation/official summaries; **do not invent rate tables**.

**Primary URLs (explainers / official)**  
- https://eur-lex.europa.eu (search Interchange Fee Regulation / Regulation (EU) 2015/751)  
- https://stripe.com/resources/more/merchant-discount-rate  
- https://www.checkout.com/blog/merchant-discount-rate  
- https://www.jpmorgan.com/content/dam/jpm/merchant-services/documents/jpmorgan-interchange-guide.pdf  

---

### RegTech beyond KYC (Travel Rule / sanctions)

**What it is**  
Continuous **sanctions/PEP** screening on payments, payment-fraud data sharing, and—for crypto/CASP corridors—**Travel Rule** originator/beneficiary information exchange (FATF recommendations; local implementations). Relevant as Banking Circle publicly positions **CSSF CASP** + stablecoin settlement.

**How it differs**  
KYC onboarding ≠ payment-time screening ≠ Travel Rule messaging between VASPs/CASPs.

**Example companies / systems**  
ComplyAdvantage; Notabene (Travel Rule); Chainalysis / Elliptic (crypto compliance—confirm programme fit); OFSI (UK sanctions authority); FATF overview.

**Primary URLs**  
- https://www.fatf-gafi.org  
- https://notabene.id  
- https://complyadvantage.com  
- https://www.gov.uk/government/organisations/office-of-financial-sanctions-implementation  
- https://www.bankingcircle.com/banking-circle-introduces-stablecoin-settlement-services/  

---

### Core banking / processor platforms (brief)

**What it is**  
The **middleware** many BaaS/EMI/bank stacks run: core banking ledgers/product processors, issuer processors, and acquiring hosts—below the customer-facing API brand.

**How it differs**  
Thought Machine / Mambu / Temenos ≠ Banking Circle (licence + rails). Thredd / Marqeta are issuer processors already listed under issuing—repeated here as the “platform” layer.

**Examples**  
Thought Machine, Mambu, Temenos; Thredd; Marqeta; (US) FIS Issuer Solutions / TSYS context after Global Payments–FIS 2026 asset swap—confirm current branding per programme.

**Primary URLs**  
- https://www.thoughtmachine.net  
- https://www.mambu.com  
- https://www.temenos.com  
- https://www.thredd.ai  
- https://www.marqeta.com  

---

## Adjacent verticals (brief; out of deep scope)

**Lending / BNPL** — Checkout-adjacent consumer credit or merchant-funded instalments; EU/UK consumer-credit rules apply. Names: Klarna, Affirm, PayPal Pay in N. Not required for a Banking Circle + B4B merchant-payout stitch unless checkout partners demand it.  
https://www.klarna.com · https://www.affirm.com

**Wealth / investments** — Brokerage, robo-advisory, and fund platforms sit beside payments but use different licences (MiFID/RAO etc.). Out of deep scope here; only note when treasury/sweep products touch your ledger.

**Insurance / insurtech** — Explicitly **omitted** from this payments-integrator map (bancassurance is adjacent, not part of collect→hold→move→recon).

**APAC / LATAM A2A rails (corridor awareness)** — UPI (India), PIX (Brazil), PayNow (Singapore) matter when FX/payout vendors (Wise, Currencycloud, Airwallex) advertise local account details.  
https://www.npci.org.in · https://www.bcb.gov.br · https://www.mas.gov.sg

---

## How B4B + Banking Circle fit together

| | **Banking Circle** | **B4B Payments** |
| --- | --- | --- |
| Public role | B2B **credit institution** (CSSF); payments, accounts, FX, VIBANs, clearing/correspondent; Connect APIs; CASP/stablecoin settlement (announced) | **EMI** (UK FCA + LT BoL); accounts, payments, FX, **cards**, spend; embedded/partner programmes |
| Customers | Regulated FIs, PSPs, banks, funds | Businesses + platforms/partners; programmes needing cards/e-money |
| Client money | Bank / CI regime (deposit-side protections where eligible) | EMI **safeguarding** (not FSCS on EMI balance) |
| Cards | Not the consumer card programme issuer in public positioning | Core: Mastercard principal UK/EU; US Visa via Cross River |
| Group | Parent/ecosystem bank | Independent **sister** in Banking Circle Group (acquisition announced 2021; operates as sister company) |
| Typical handoff | Safeguarding accounts / rails / FX behind the scenes | Regulated customer/programme face; KYC/AML/scheme; optional path to **Banking Circle Direct** at scale |

**Integrator takeaway:** Use **B4B** when you need **EMI + cards/BIN sponsorship + partner licensing** across UK/EU (and US card via Cross River). Use **Banking Circle** when you are (or become) a regulated FI needing **wholesale bank rails**, VIBANs, FX, and settlement APIs. Many stacks use **both**: B4B as the regulated programme/EMI layer, Banking Circle as the bank infrastructure—consistent with B4B’s “Powered by Banking Circle” positioning. Partner **Oversight** API (password-gated ReadMe) sits on the pre-settlement control path before Banking Circle settlement.

---

## Typical merchant-payout stack (example)

```
Buyer pays merchant/marketplace
        │
        ├──[card]──► [1a] Acquirer / gateway     e.g. Adyen, Stripe, Checkout.com
        │                    │ settlement to platform float
        │                    ▼
        └──[A2A]──► [1b] Pay-by-bank (PISP)     e.g. TrueLayer → ASPSP / FPS/SCT Inst
                             │
                             ▼
                    [2] EMI / programme + Oversight  e.g. B4B (safeguarding, KYC/KYB,
                             │  optional payout cards, partner licence;
                             │  Oversight API where contracted)
                             ▼
                    [3] Bank rail / VIBAN / FX      e.g. Banking Circle (SEPA/FPS/SWIFT,
                             │  multi-currency, Connect status APIs)
                             ▼
                    [4] Seller disbursement         bank account credit and/or payout card
                             │
                             ▼
                    [5] Recon & ledger              webhooks/ISO status + internal ledger
                                                   (Modern Treasury / Formance / Atlar)
```

**Side modules (almost always):** identity/KYB vendor + fraud/TM vendor; card programmes add **issuer processor** (e.g. Thredd), **3DS/token** services, **scheme** rules via BIN sponsor, and **dispute** ops (Verifi/Ethoca + acquirer tools).

**InfinitePay-style vignette (centre of this map):** Marketplace or merchant platform collects via acquirer (or pay-by-bank), holds programme balances under an **EMI** (B4B-class) with safeguarding and optional payout cards, moves multi-currency/SEPA/FPS value on **Banking Circle** rails/VIBANs, and reconciles Connect/ISO status into an internal ledger—control plane shared across both vendors.

---

## Source anchors (primary)

**Regulators / registers**  
- FCA Financial Services Register: https://www.fca.org.uk/firms/financial-services-register  
- FCA using PSPs (safeguarding vs FSCS): https://www.fca.org.uk/consumers/using-payment-service-providers  
- FCA EMI/PI safeguarding: https://www.fca.org.uk/firms/emi-payment-institutions-safeguarding-requirements  
- FCA PS25/12 safeguarding regime: https://www.fca.org.uk/publications/policy-statements/ps25-12-changes-safeguarding-regime-payments-and-e-money-firms  
- FSCS e-money note: https://www.fscs.org.uk/news/protection/e-money-and-fscs-protection/  
- CSSF: https://www.cssf.lu  
- Bank of Lithuania: https://www.lb.lt  
- BaFin: https://www.bafin.de  
- EUR-Lex PSD2: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32015L2366  

**Market infrastructure**  
- EPC (SCT / SCT Inst / SDD / SRTP): https://www.europeanpaymentscouncil.eu  
- ECB TARGET / T2 / TIPS: https://www.ecb.europa.eu/paym/target/html/index.en.html · https://www.ecb.europa.eu/paym/target/tips/html/index.en.html  
- EBA Clearing: https://www.ebaclearing.eu  
- Pay.UK: https://www.wearepay.uk/  
- Fedwire / FedACH / FedNow: https://www.federalreserve.gov/paymentsystems/fedfunds_about.htm · https://www.frbservices.org/financial-services/ach · https://www.frbservices.org/financial-services/fednow  
- The Clearing House RTP: https://www.theclearinghouse.org/payment-systems/rtp  
- ISO 20022 catalogue (incl. pacs.008): https://www.iso20022.org/catalogue-messages  
- SWIFT ISO 20022 for Dummies (public PDF): https://www.swift.com/sites/default/files/files/swift-iso20022fordummies-6thedition-2022.pdf  
- BIS CPMI: https://www.bis.org/cpmi/  

**Schemes / auth / disputes**  
- EMVCo 3DS: https://www.emvco.com/emv-technologies/3-d-secure/  
- Visa Token Service: https://usa.visa.com/products/visa-token-service.html  
- Mastercard MDES docs: https://developer.mastercard.com/mdes-digital-enablement/documentation/  
- Verifi RDR: https://www.verifi.com/rdr-automatically-stop-chargebacks.html  
- Ethoca: https://www.ethoca.com  
- Lithic transaction flow / settlement: https://docs.lithic.com/docs/transaction-flow · https://www.lithic.com/blog/what-is-settlement-processing  

**Core vendors in this map**  
- Banking Circle regulatory / home / stablecoin CASP news: https://www.bankingcircle.com/regulatory-information/ · https://www.bankingcircle.com · https://www.bankingcircle.com/banking-circle-introduces-stablecoin-settlement-services/  
- B4B regulatory / partners / ReadMe (password gate): https://www.b4bpayments.com/c/regulatory · https://www.b4bpayments.com/c/solutions/partners · https://b4bpayments.readme.io  
- Equals (ex-Railsr) rebrand: https://equalsmoney.com/newsroom/equals-money-railsr-becomes-equals  
- Visa completes Currencycloud acquisition: https://usa.visa.com/about-visa/newsroom/press-releases.releaseId.18666.html  
- TrueLayer security (AIS/PIS + EMI FRN 901096): https://truelayer.com/security/security-at-truelayer/  
- PFS T&Cs (FRN 900036): https://prepaidfinancialservices.com/terms-conditions  
- Paysera–Contis acquisition: https://www.paysera.com/v2/en/blog/paysera-acquisition-of-contis-approved  
- Adyen licences EMEA: https://www.adyen.com/licenses/emea
- Adyen N.V. DNB register: https://www.dnb.nl/en/public-register/information-detail/?registerCode=WFTDG&relationNumber=F0001
- Global Payments Worldpay Form 8-K (completed 9 Jan 2026; PR 12 Jan): https://investors.globalpayments.com/financial-information/all-sec-filings/content/0001104659-26-002705/tm262856d1_8k.htm
- Global Payments completes Worldpay acquisition: https://investors.globalpayments.com/news-events/press-releases/detail/498/global-payments-completes-acquisition-of-worldpay-and  
- ClearBank, Solaris licence, Circle, Stripe/Adyen docs — as linked in sections above  

**Internal**  
- Study pack: https://github.com/webduvet/avram/tree/main/docs/study-material  

*As of 2026-09-08 this map reflects public positioning on the linked pages; re-check the FCA/CSSF/BoL/BaFin register (and vendor legal pages) before contract. Educational only—not legal advice.*

---

## Verification notes

Spot-check (`curl -sI -L --max-time 15`) of critical primaries on **2026-09-08** (Europe/Dublin). Soft blocks / password gates already labelled in the body are unchanged.

| URL | Status |
| --- | --- |
| https://www.fca.org.uk/consumers/using-payment-service-providers | 200 |
| https://www.fscs.org.uk/news/protection/e-money-and-fscs-protection/ | 200 |
| https://www.fca.org.uk/firms/emi-payment-institutions-safeguarding-requirements | 200 |
| https://www.fca.org.uk/publications/policy-statements/ps25-12-changes-safeguarding-regime-payments-and-e-money-firms | 200 |
| https://www.bankingcircle.com/regulatory-information/ | 200 |
| https://www.b4bpayments.com/c/regulatory | 200 |
| https://www.adyen.com/licenses/emea | 200 |
| https://www.dnb.nl/en/public-register/information-detail/?registerCode=WFTDG&relationNumber=F0001 | 200 |
| https://equalsmoney.com/newsroom/equals-money-railsr-becomes-equals | 200 |
| https://truelayer.com/security/security-at-truelayer/ | 200 |
| https://www.solarisgroup.com/en/license/ | 200 |
| https://clear.bank/ | 200 |
| https://www.emvco.com/emv-technologies/3-d-secure/ | 200 |
| https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32015L2366 | 200 |
| https://www.europeanpaymentscouncil.eu/what-we-do/epc-payment-schemes/sepa-instant-credit-transfer/sepa-instant-credit-transfer-rulebook | 200 |
| https://www.ecb.europa.eu/paym/target/tips/html/index.en.html | 200 |
| https://www.wearepay.uk/ | 200 |
| https://investors.globalpayments.com/news-events/press-releases/detail/498/global-payments-completes-acquisition-of-worldpay-and | 200 |
| https://www.bankingcircle.com/banking-circle-introduces-stablecoin-settlement-services/ | 200 |
| https://www.iso20022.org/catalogue-messages | **hard-fail** — TLS connects then hangs; no HTTP status (timeout / 0 bytes); URL left as in source; no replacement invented |

Access-gated (already labelled in source; not treated as hard-fail): `docs.bankingcircleconnect.com`, `b4bpayments.readme.io`.
