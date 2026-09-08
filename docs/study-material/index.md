# Study material

Verified study links for payments / Sim Lab. Catalogue compiled **2026-09-08**. Prefer free / low-cost. Only URLs confirmed via WebSearch and/or WebFetch/curl (HTTP 200 or indexed official landing pages).

**Verification notes**

- Most links returned HTTP 200 from the verification environment.
- `iso20022.org` and `swift.com` are blocked from some egress (connection fail) but return clear, indexed official landing pages via WebSearch — included as verified public pages.
- Full Mastercard IPM / Visa BASE II manuals are **members-only**; only public overviews listed.

Related: [Sim Lab simulation contracts](../fintech/simulation/index.md).

## Pages

- [Foundations — central bank / payment systems](foundations.md)
- [Card rails & clearing formats](card-rails-clearing.md)
- [ISO 20022 & SEPA](iso20022-sepa.md)
- [Ledgers & accounting engines](ledgers.md)
- [Engineering blogs / open infra](engineering-oss.md)
- [Discrete-event simulation](discrete-event-simulation.md)
- [Courses](courses.md)

## Sim Lab hook (Auth → Capture → Clearing → Settlement → Payout)

| Stage | Typical protocols / messages | What to simulate (conceptual) |
| --- | --- | --- |
| **Auth** | ISO 8583 MTI 0100/0110 (DMS); EMV ARQC in DE 55; ASA-style decision latency | Timeout budgets, stand-in advice, partial approval, balance holds (`pending` ledger), cryptogram pass/fail |
| **Capture** | Acquirer completion / advice (e.g. 0220); SMS financial auth (auth+capture combined) | Under/over-capture, multi-completion, auth expiry vs reversal races |
| **Clearing** | Scheme files: Mastercard **IPM**/GCMS; Visa **BASE II**; ISO 20022 pacs for A2A; ACH batch files (NACHA) | File cycles, FX conversion, interchange fee calc, unmatched / force-post exceptions |
| **Settlement** | RTGS: **Fedwire / T2**; Instant: **FedNow / TIPS / SCT Inst**; ACH deferred net; scheme net settlement | Cutoff calendars, net vs gross, finality, liquidity shortfalls, multilateral netting |
| **Payout** | ACH credit, RTP/FedNow, wire, push-to-card; ledger post from FBO/omnibus | Available vs posted balances, returns (R01…), webhook/ledger cascades, idempotent payouts |

**Ledger mapping tip:** Map Auth→`pending` transfer, Capture/Clearing confirmation→`post_pending`, Expiry/Reversal→`void_pending` (TigerBeetle two-phase or Modern Treasury pending transactions).

## Reading order (suggested 2-week path)

**Week 1 — Rails & messages**

1. Day 1–2: Fedwire / FedACH / FedNow primers + FedNow Technical Overview PDF; ECB T2 + TIPS pages.
2. Day 3: BIS Red Book methodology skim + BIS Data Portal browse (one country LVPS/FPS).
3. Day 4–5: Lithic Transaction Flow + Authorization Platform blog; ISO 8583 DMS vs SMS.
4. Day 6: ISO 20022 for Dummies + iso20022.org pacs.008 catalogue; EPC SCT Inst landing + skim IG TOC.
5. Day 7: Kleppmann accounting essay + TigerBeetle two-phase transfers.

**Week 2 — Ledger, files, simulation**

6. Day 8–9: Modern Treasury Scale-a-Ledger parts II/IV/V + FBO/wallet posts; Fowler Accounting Transaction.
7. Day 10: Moov ACH docs + clone `moov-io/ach`; skim Silverflow/Intelica clearing overviews (note members-only manuals).
8. Day 11: EMVCo overview + AWS ARQC page + CorebaseIT cryptogram guide (high-level only).
9. Day 12–13: SimPy tutorial; FoundationDB simulation docs + Zemb deep dive; decide DES event model for cutoffs/timeouts.
10. Day 14: Optional Coursera payment/settlement audit modules; sketch Sim Lab hook table as executable scenarios (auth hold → clear → settle → payout).

## Caveats

1. **Scheme manuals (Mastercard IPM/GCMS, Visa BASE II/SMS, full ISO 8583 network dialects)** are **members-only / licensed**. Public blogs and processor docs are overviews only — do not treat them as authoritative field dictionaries.
2. **EMVCo Book 2 / full chip specs** often require free registration or Associate access; use AWS/CorebaseIT for conceptual ARQC only.
3. **SWIFT Smart** is not a public MOOC — access is for SWIFT-connected institutions; paid instructor-led/cert exams are separate.
4. **iso20022.org / swift.com** may be firewalled from some networks; use catalogue search + MyStandards (FedNow mentions free MyStandards registration) as alternate access paths.
5. **Banks DES book** — purchase or library; no pirate links.
6. **Coursera/edX “free”** usually means audit (videos/readings); graded work & certificates cost money.
7. **Payments Academy** courses are free; certification maintenance is **$49/year** after you pass.
8. **PCN** training brand was not cleanly verified — treat as unresolved; don’t budget against an invented product.
9. **Form3 / Volante** free deep whitepapers were not solidly verified this pass — gap for a follow-up if vendor unlocks public PDFs.
10. Prefer **central-bank / EPC / BIS / Fed** primaries over vendor marketing when modeling finality, cutoffs, and legal settlement.
