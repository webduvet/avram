# Engineering blogs / open infra

- [Lithic blog — Card authorization platform](https://www.lithic.com/blog/card-authorization-platform) — **free** — Multi-network ISO 8583 normalization & HA auth architecture.
- [Lithic blog — Settlement processing](https://www.lithic.com/blog/what-is-settlement-processing) — **free** — Clearing files → net settlement → reconciliation ops.
- [Lithic docs — Transaction flow](https://docs.lithic.com/docs/transaction-flow) — **free** — Event-level auth/clearing/reversal state machine for sim scenarios.
- [Moov docs — ACH transfers](https://docs.moov.io/guides/money-movement/accept-payments/ach/) — **free** — ACH credit/debit flows, cutoffs, returns in a modern API context.
- [moov-io/ach (GitHub)](https://github.com/moov-io/ach) — **free (OSS)** — NACHA ACH file reader/writer/validator — ideal for ACH batch sims.
- [moov-io/iso8583 (GitHub)](https://github.com/moov-io/iso8583) — **free (OSS)** — Go ISO 8583 encode/decode library for building synthetic network messages.
- [Silverflow — File subscriptions](https://docs.silverflow.com/guides/file-subscriptions) — **free** — Modern processor view of IPM / BASE II clearing file consumption.
- [Intelica — Scheme clearing overview](https://www.intelica.com/en/blog/mastercard-visa-clearing-process) — **free** — Secondary architecture overview of scheme clearing cycles.

*Gap:* Form3 / Volante solid free whitepapers were not verified as freely downloadable technical deep-dives in this pass (vendor sites often gated); prefer Lithic + Moov + Fed/ECB primaries instead.

Lithic / Silverflow / Intelica substance also appears under [Card rails & clearing formats](card-rails-clearing.md).
