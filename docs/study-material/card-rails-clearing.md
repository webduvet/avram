# Card rails & clearing formats

- [Lithic docs — Transaction Flow (DMS vs SMS)](https://docs.lithic.com/docs/transaction-flow) — **free** — Excellent public explainer of dual-message (auth→clearing) vs single-message financial auth, holds, expiry, force posts.
- [Lithic — Architecting a Modern Card Authorization Platform](https://www.lithic.com/blog/card-authorization-platform) — **free** — How ISO 8583 variants differ by network and why edge normalization matters for multi-network processors.
- [Lithic — Guide to Settlement Processing for Card Programs](https://www.lithic.com/blog/what-is-settlement-processing) — **free** — Auth vs clearing vs settlement timelines; net settlement, interchange, reconciliation challenges.
- [Intelica — Mastercard & Visa clearing process](https://www.intelica.com/en/blog/mastercard-visa-clearing-process) — **free (reputable secondary)** — Public overview of Visa BASE II vs Mastercard GCMS/IPM; notes that full manuals are scheme members-only.
- [Silverflow docs — File subscriptions (IPM / BASE II)](https://docs.silverflow.com/guides/file-subscriptions) — **free (processor docs)** — Concise definitions of `mastercard_ipm` and `visa_base2` file types used in clearing/settlement.
- [EMVCo — Overview](https://www.emvco.com/about-us/overview-of-emvco/) — **free** — Specs are royalty-free; many detailed PDFs need free/public membership login.
- [EMVCo — Contact Chip technologies](https://www.emvco.com/emv-technologies/contact/) — **free landing** — Entry point into EMV Contact Chip specs (download often behind account).
- [AWS Payment Cryptography — Verify EMV ARQC / ARPC](https://docs.aws.amazon.com/payment-cryptography/latest/userguide/use-cases-issuers.generalfunctions.arqc.html) — **free** — High-level public ARQC validation model (session keys, CSK, ARPC) without needing scheme manuals.
- [CorebaseIT — ARQC, TC, and AAC field guide](https://corebaseit.com/corebaseit_posts/arqc-and-company/) — **free** — Practical taxonomy of EMV application cryptograms and DE 55 / tags 9F26–9F27.

Also listed under [Engineering blogs / open infra](engineering-oss.md) where the source duplicates Lithic / Silverflow / Intelica links.
