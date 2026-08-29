# Integrator baseline

Research snapshot as of 2026-08-29. Primary sources unless flagged. No personal information. Subject is **B4B Payments** (card issuing / BIN sponsorship / prepaid EMI), not other “B4B” name collisions.

Official site: [https://www.b4bpayments.com/c](https://www.b4bpayments.com/c)

## Disambiguation

The relevant firm in a payments / Banking Circle context is **B4B Payments**, the trading name of:

- **Payment Card Solutions (UK) Limited** (Companies House **05941947**) — UK EMI
- **UAB B4B Payments Europe** (Lithuanian company code **305539054**, Bank of Lithuania authorisation **LB002020**, EMI licence **76**) — EU EMI
- **B4B Payments (USA) Inc.** — US programme manager (not the US issuer)

**Other “B4B” name collisions (not this subject):**

| Name | Why it is not the subject |
|---|---|
| **PCS B4B LTD** (CH **11220347**), previously **B4B PAYMENTS LTD** (22 Feb 2018 – 29 Apr 2020) | Dissolved 3 Nov 2020. Holding-company SIC, not the operating EMI. [Companies House](https://find-and-update.company-information.service.gov.uk/company/11220347) |
| **B4B UK LTD** (SC106108) | Separate Scottish company; no evidence it is this EMI. [Companies House](https://find-and-update.company-information.service.gov.uk/company/SC106108) |
| Generic “B4B” / business-to-business | Overloaded acronym; ignore unless it maps to b4bpayments.com |

No second plausible payments EMI / BIN-sponsor / card-issuing firm using the B4B name was found on official regulator or company-register sources.

## Product map

```
                         Banking Circle Group (bank + rails + FX)
                                      |
                    +-----------------+------------------+
                    |                                    |
         B4B Payments (EMI / issuing)          Banking Circle Direct
         UK EMI + LT EMI + US PM               (graduation path for
                                               scaled licensed EMIs)
                    |
     +--------------+------------------+------------------+
     |              |                  |                  |
 Direct B2B     Platform /         BIN sponsor        Settlement-only
 (own spend,    embedded           (B4B scheme        (partner has own
  accounts,     finance            + licence          scheme; B4B
  cards)        (white-label)      + processor)       settlement/rails)
     |
     +-- Cards: prepaid MC/Visa (UK/EU); Visa prepaid via Cross River (US)
     |     physical | virtual | wallet (Apple Pay / Google Pay, 3DS, tokenised)
     |     debit-BIN marketing; legal form = prepaid e-money, not debit/credit
     +-- Accounts / wallets: e-money accounts, VIBANs, 7 held CCY
     +-- Payments: FPS, SEPA/SEPA Instant, SWIFT, ACH, CoP/VoP
     +-- FX: spot + 24h held rates; 21 CCY via SWIFT
     +-- Spend: MCC/geo/channel limits, receipts, approvals
     +-- APIs (gated): Cards, Accounts, Payments, FX, webhooks, partner KYB pack
     +-- Processor named: Thredd
```

**Prepaid vs debit vs credit (as documented):** prepaid e-money cards, including some issued on debit BINs per marketing. Explicitly **not** debit, charge, or credit in UK/US cardholder T&Cs. No credit-issuing product found on official pages.

## Findings (sourced)

- **UK EMI, not a bank.** Payment Card Solutions (UK) Limited (CH **05941947**, incorporated 20 Sep 2006, active, 12-18 Grosvenor Gardens, London SW1W 0DH) trades as B4B Payments and is stated by the firm to be authorised by the FCA under the Electronic Money Regulations 2011, **FRN 930619**. UK T&Cs confirm the same FRN; funds are safeguarded with a regulated credit institution, **not FSCS-protected**. Live FCA Register page was not independently readable (JS/Cloudflare); treat FRN as firm-stated pending a live register check.  
  Sources: [Regulatory](https://www.b4bpayments.com/c/regulatory) · [CH 05941947](https://find-and-update.company-information.service.gov.uk/company/05941947) · [Security / FSCS](https://www.b4bpayments.com/c/security) · [UK Enterprise T&Cs PDF](https://www.b4bpayments.com/c/api/media/file/B4B-UK-Terms-and-Conditions-of-Business-Enterprise-Version-2.0.pdf)

- **EU EMI (Lithuania), passporting without a branch.** UAB B4B Payments Europe (code **305539054**, Didžioji g. 18, LT-01128 Vilnius) holds Bank of Lithuania **Electronic money institution licence No. 76**, valid from **2020-11-26**, authorisation **LB002020**. Activities include issuing, distributing and redeeming e-money; payment accounts / cash withdrawals; payment transactions (direct debits, card or similar, credit transfers); issuing payment instruments and/or acquiring. BoL records **provision of services without a branch in other Member States (29)**. EU T&Cs v2.0 (August 2025) restate licence 76 / LB002020.  
  Sources: [BoL participant](https://www.lb.lt/en/sfi-financial-market-participants/uab-b4b-payments-europe) · [BoL licence 76](https://www.lb.lt/en/licences-1/view_license?id=1995) · [EU T&Cs PDF](https://www.b4bpayments.com/c/api/media/file/B4B-Europe-Terms-and-Conditions-of-Business-Version-2.0.pdf)

- **US is programme-managed prepaid, issued by a bank partner — not a B4B US bank/EMI licence.** “The B4B Payments Visa® prepaid card is issued by Cross River Bank, Member FDIC pursuant to a license from Visa U.S.A. Inc.” US cardholder T&Cs name **B4B Payments (USA) Inc.** as prepaid **program manager**. US product pages: **Visa prepaid only** (not Mastercard) in USD. Privacy policy: B4B Payments (USA) Inc., company number **7529211**, 3 Allied Drive, Suite 303, Dedham, MA 02026.  
  Sources: [Regulatory](https://www.b4bpayments.com/c/regulatory) · [US Cards](https://www.b4bpayments.com/c/us/products/cards) · [US Corporate Cardholder Terms PDF](https://www.b4bpayments.com/c/api/media/file/B4B-USA-%20Corporate-Cardholder-Terms-8072026.pdf) · [Privacy](https://www.b4bpayments.com/c/privacy)

- **Group holding company and Banking Circle relationship.** Immediate UK parent of the EMI is **Payment Card Solutions Group Limited** (CH **10149512**, 75%+ of 05941947). After 29 Dec 2022 the Group company has **no registrable PSC**. B4B About: became part of the Banking Circle Group in 2020–2021. Banking Circle 23 Nov 2021: after closing, B4B “will operate as an independent **sister company** of Banking Circle.” 30 May 2022 ecosystem article lists B4B among current applications (card issuing and expense management). Companies House does **not** name Banking Circle as a PSC of the Group company.  
  Sources: [CH PSC 05941947](https://find-and-update.company-information.service.gov.uk/company/05941947/persons-with-significant-control) · [CH 10149512](https://find-and-update.company-information.service.gov.uk/company/10149512) · [CH PSC 10149512](https://find-and-update.company-information.service.gov.uk/company/10149512/persons-with-significant-control) · [About](https://www.b4bpayments.com/c/about) · [BC 2021 announcement](https://www.bankingcircle.com/b4b-payments-joins-banking-circle-ecosystem/) · [BC 2022 ecosystem](https://www.bankingcircle.com/banking-circle-ecosystem-tackles-rebundling-of-financial-services/)

- **Who they sell to: dual GTM — direct businesses + platform/BIN-sponsor partners; US role is programme manager.** Direct: businesses issue prepaid cards / open e-money accounts without a lengthy integration. Partners: platforms, SaaS, fintechs, marketplaces. Partnership models: **BIN Sponsorship**, **Settlement Only** (regulated EMI/PI with own scheme relationship), **Full Scheme Membership Support** (Banking Circle banking backbone), and **Banking Circle Direct** as a graduation path. Partners “may be able to operate under” B4B’s UK/EU e-money licences (subject to model, regulatory requirements, programme approval).  
  Sources: [Platform Partners](https://www.b4bpayments.com/c/solutions/partners) · [Cards](https://www.b4bpayments.com/c/products/cards) · [Platform API](https://www.b4bpayments.com/c/products/platform-api) · [US Corporate Cardholder Terms PDF](https://www.b4bpayments.com/c/api/media/file/B4B-USA-%20Corporate-Cardholder-Terms-8072026.pdf)

- **Card product is prepaid e-money (physical/virtual/wallet); marketing “debit BIN” does not make it a debit or credit card.** UK Card User Terms: “The Card is a prepaid Card. It is not a debit card and is not connected to any bank account. It is also not a guarantee card, a charge card or a credit card.” US T&Cs: prepaid Visa; not a debit card; not a credit card. Marketing still says “Debit BINs with prepaid funding” and dual-scheme Mastercard + Visa in UK/EU; US is Visa via Cross River. ATM cash-out is **not default** on UK cards unless the Business Partner has permitted it. No revolving-credit product is described.  
  Sources: [UK Corporate Card User Terms PDF](https://www.b4bpayments.com/c/api/media/file/B4B-UK-Corporate-Card-User-Terms-Enterprise.pdf) · [US Consumer Cardholder Terms PDF](https://www.b4bpayments.com/c/api/media/file/B4B%20USA%20Consumer%20Cardholder%20Terms_Final_08192026.pdf) · [Cards](https://www.b4bpayments.com/c/products/cards)

- **Scheme membership, processor, wallets, 3DS, tokenisation.** About: **Mastercard Principal Member (UK and EU)**; **Visa Partner (UK, EU and US)**. Partners/BIN page is narrower on Visa: “Mastercard Principal Member (UK and EU) and Visa Partner (US).” Web/mobile and spend pages separately say “Principal member of both networks” — **inconsistent with About**; do not treat the latter as confirmed. Processor: **Thredd**. Tokenised cards for Apple Pay and Google Pay; 3D Secure for all card transactions; full PANs never stored on the B4B platform; PCI DSS; Cyber Essentials Plus.  
  Sources: [About](https://www.b4bpayments.com/c/about) · [Platform Partners](https://www.b4bpayments.com/c/solutions/partners) · [Security](https://www.b4bpayments.com/c/security) · [Web & Mobile](https://www.b4bpayments.com/c/products/web-mobile-app)

- **Accounts, payouts, FX, multi-currency — powered by Banking Circle rails.** Held currencies: **GBP, EUR, USD, SEK, DKK, AED, AUD (7)**. Pay-out via SWIFT FX to **21 currencies**. Rails named: Faster Payments, SEPA, SEPA Instant, SWIFT, ACH; local rails in Denmark, Australia “and other markets” via Banking Circle. Structure: master account → entity accounts → virtual accounts / VIBANs. UK CoP and SEPA VoP are marketed. Open Banking on that product page is **read-only** reconciliation, not payment initiation.  
  Sources: [Business Accounts](https://www.b4bpayments.com/c/products/business-accounts) · [Payments & FX](https://www.b4bpayments.com/c/products/payments-fx)

- **API surface is sales-gated, not a public developer portal.** JSON REST; **OAuth 2.0 customer-credentials flow**; tokens **valid 60 minutes** with programmatic refresh; signed webhooks; sandbox that “mirrors production APIs with test credentials.” Named families: **Cards**, **Accounts**, **Payments**, **FX**. Partner API also: onboarding documents and identity data; cardholders; beneficiaries; VIBANs; compliance documents; **health-check endpoint**. Full guides, schemas, and sandbox credentials are **available on request** — not published. `docs.b4bpayments.com` does not resolve; `api.b4bpayments.com` and `app.b4bpayments.com` return HTTP 403; `sandbox.b4bpayments.com` redirects to the marketing site. Separate **PSD2 dedicated interface** for AISP/PISP/CBPII (eIDAS QWAC + QSealC) is also on-request.  
  Sources: [Developers](https://www.b4bpayments.com/c/developers) · [Platform API](https://www.b4bpayments.com/c/products/platform-api) · [TPP / PSD2](https://www.b4bpayments.com/c/tpp)

- **White-label, BIN sponsorship caveats, and risk-appetite prohibitions.** White-label: co-branded or fully custom cards; app branding “for larger programmes”; cards in EUR/GBP/USD. BIN sponsorship is a **bridge** while EMI/PI scheme membership progresses, “subject to your specific licence, permissions, and scheme approval.” Processor/scheme/regulatory stack stays with B4B. Risk Appetite Statement (last updated December 2023) **prohibits** (among others): sanctioned parties; shell banks/companies; bearer-share / nominee concealment; physically present adult services; unregulated/offshore investment companies; true holding companies with no operations; binary options; weapons of war/defence equipment; **virtual currencies / crypto-asset firms**; **gambling/gaming**; **CBD**. Restricted (high-risk, extra DD, often UK/EEA-only): non-physical adult content, affiliate/MLM, charities/NPOs, CSPs, crowdfunding, pharma, HVGs, recreational weapons, licensed financial-services firms, marketplaces (5+ year track record), import/export. Spend-management product also exposes **MCC, channel, geographic, and ATM** controls.  
  Sources: [Platform Partners](https://www.b4bpayments.com/c/solutions/partners) · [Risk Appetite](https://www.b4bpayments.com/c/risk-appetite) · [Spend Management](https://www.b4bpayments.com/c/products/spend-management)

- **Safeguarding, not deposit insurance.** Customer funds are **not FSCS-protected**. UK funds safeguarded under EMR 2011 in segregated accounts with authorised credit institutions; EU funds under Lithuanian e-money law.  
  Sources: [Security](https://www.b4bpayments.com/c/security) · [Safeguarding Policy PDF](https://www.b4bpayments.com/c/api/media/file/B4B-Payments_Safeguarding-Policy-.pdf)

- **Mastercard BIN Sponsor Plus (2026) is claimed on Mastercard/B4B social, not on B4B’s own regulatory page.** Mastercard’s January 2026 BIN Sponsor Plus launch PR named four founding UK sponsors — **not** B4B. A later Mastercard LinkedIn post names B4B as “the first new BIN Sponsor Plus partner since launch.” Not independently confirmed on mastercard.com newsroom or b4bpayments.com regulatory copy. Treat as unofficial until a durable primary page confirms it.  
  Source: [Mastercard launch PR](https://mastercardcontentexchange.com/news/europe/en-uk/newsroom/press-releases/en-gb/mastercard-launches-best-in-class-bin-sponsor-plus-programme-to-support-next-generation-of-u-k-fintechs/) (founders only)

## API / docs URLs actually opened

| URL | What it is | Public? |
|---|---|---|
| [Developers](https://www.b4bpayments.com/c/developers) | Marketing developer overview (auth, sandbox, API families) | Yes |
| [US Developers](https://www.b4bpayments.com/c/us/developers) | US-locale copy of the same | Yes |
| [Platform API](https://www.b4bpayments.com/c/products/platform-api) | Product-level API scope + partner/BIN methods | Yes |
| [TPP / PSD2](https://www.b4bpayments.com/c/tpp) | PSD2 dedicated interface | Yes (docs on request) |
| `https://docs.b4bpayments.com` | Does **not** resolve (DNS) | No public portal found |
| `https://api.b4bpayments.com` / `https://app.b4bpayments.com` | HTTP 403 | Login/walled |
| `https://sandbox.b4bpayments.com` | 301 → marketing homepage | Not a self-serve sandbox |

**No OpenAPI/Swagger, endpoint paths, request schemas, webhook event catalogue, or sandbox signup were publicly available.**

## Open questions

1. **Live FCA Register record for FRN 930619** — permissions, status, agents, passporting. Firm-stated EMI authorisation only until a live register pull succeeds.
2. **Full public API contract** — base URLs, endpoint list, auth token URL, webhook event names, idempotency, rate limits, PCI/PAN handling via API, 3DS challenge flow, Apple/Google Pay in-app provisioning API. All login-walled / on-request.
3. **Sandbox** — existence is claimed; no self-serve URL or test-card catalogue is published.
4. **Scheme membership precision** — Mastercard Principal (UK/EU) is consistent; Visa is Partner (UK+EU+US) on About vs Partner (US) on the BIN page vs “principal member of both” on spend/web pages.
5. **Whether UK/EU issuing uses debit BINs in scheme terms** vs purely prepaid BINs. Marketing vs T&Cs disagree on the word “debit”.
6. **Credit products** — none documented. Cannot assert they will never offer credit.
7. **Thredd** — named as processor; commercial scope not specified.
8. **Lithuania passporting country list** — BoL says 29 Member States without a branch; per-country table not extracted.
9. **US entity registration** — privacy policy gives 7529211; Delaware/Massachusetts filing not opened. No US money-transmitter licences found on b4bpayments.com (issuing sits with Cross River).
10. **Banking Circle legal ownership of PCS Group** — commercially asserted; UK PSC register shows “no registrable person”.
11. **Mastercard BIN Sponsor Plus** — LinkedIn after the Jan 2026 launch PR; not on B4B’s own regulatory page.
12. **Agent / distributor register** — public list of appointed agents/distributors not found.
13. **Geographic issuance vs spend eligibility** — Risk Appetite restricts some sectors to UK/EEA; full permitted-country matrix is not public.
14. **KYC vendor in the API** — privacy policy names iDenfy (EU) and Equifax (UK); whether those are API steps or ops is not documented.
15. **Sitemap** `https://www.b4bpayments.com/c/sitemap.xml` returned HTTP 500 when fetched.

## Source quality

Used: B4B official site and T&C/policy PDFs; Bank of Lithuania; Companies House; Banking Circle newsroom.  
Not used as facts: blogs, LinkedIn employee counts/revenue, thebanks.eu financials.  
Mastercard BIN Sponsor Plus: launch PR used; B4B’s later addition is **not** treated as confirmed on a durable primary page.
