# Worldline

European payments group spanning **acceptance**, **acquiring**, and **issuing**. Merchant/payment-stack notes below focus on Connect + Global Collect and on using Worldline **as acquirer**.

**Docs vs marketing.** Connect developer docs are the primary source for integration and reporting facts. Global Collect pages on worldline.com are **product marketing** — useful for capability shape (cross-border ecommerce, local acquiring, multi-acquirer smart routing), not for API or file contracts.

## Connect

[Connect](https://docs.connect.worldline-solutions.com/getting-started/about-connect/) is Worldline’s integration suite onto Worldline payment platforms. Official About Connect lists supported platforms as **Global Collect** and **TechProcess**. The Connect **API Reference** concepts page also lists **Ogone** (closed beta) and **Online Payment Acceptance** as platforms reachable through the same REST surface.

What Connect offers (from About Connect / Connect home):

- REST API (Server API; Client API for dynamic checkout; Dispute API on request); idempotency to prevent duplicate transactions
- SDKs for major languages (server and client-side / native)
- MyCheckout hosted payment pages (HPP) and MyCheckout editor
- Configuration Center (self-service integration management)
- Webhooks for payment status and related events
- Payment products and features including Apple Pay, Google Pay, Worldline Account-to-Account (A2A), network tokens, dynamic 3-D Secure, plus market products (e.g. Troy, Trustly, WeChat Pay, South Korea products)
- API Explorer, API Reference, integration dashboards

Primary entry: [About Connect](https://docs.connect.worldline-solutions.com/getting-started/about-connect/) · [Connect home](https://docs.connect.worldline-solutions.com/)

## Global Collect

Cross-border ecommerce / multi-acquirer platform used with Connect. **Product marketing** (worldline.com Global Collect) positions it as a single-integration engine with local acquirer connectivity, AI/smart routing for authorisation and cost, hosted checkout / API / plugins, and consolidated reporting. See [Global Collect](https://worldline.com/en/home/main-navigation/solutions/merchants/global-collect) and [Go further in Europe](https://worldline.com/en/home/main-navigation/solutions/merchants/global-collect/go-further-in-europe-more-with-worldline) (marketing; labels full value chain acceptance / acquiring / issuing and EEA local acquiring with smart routing).

Operational recon for this stack is **not** on those marketing pages — use [Reconciliation and reporting](reconciliation-and-reporting.md).

## Other developer portals (Connect home)

Listed on the Connect documentation home for other Worldline surfaces — brief pointers only:

- **Global Online Pay (Direct)** — [docs.direct.worldline-solutions.com](https://docs.direct.worldline-solutions.com/)
- **SmartPOS | Tap on Mobile** — [docs.smartpos.worldline-solutions.com](https://docs.smartpos.worldline-solutions.com/)
- **Acquiring** — [docs.acquiring.worldline-solutions.com](https://docs.acquiring.worldline-solutions.com/)
- **TravelHub** — [docs.travel.worldline-solutions.com](https://docs.travel.worldline-solutions.com/)

## How this stack is used here

Worldline is used **as acquirer** (funds / settlement path and payment-model distinctions), not merely as a thin gateway front. Connect names two operating models — **Full Service** (Worldline collects funds into Worldline accounts and settles to the merchant bank) vs **Gateway** (technical connection + fraud only; merchant’s acquirers remit). See [Acquiring](acquiring.md).

- [Acquiring](acquiring.md) — Full Service vs Gateway operating models; FAQ funds-handling vs gateway; Global Collect local acquiring / multi-acquirer routing (marketing)
- [Reconciliation and reporting](reconciliation-and-reporting.md) — two-layer reporting; WX + Financial Report; SFT pull as the in-stack operational path

## Notes

- [Acquiring](acquiring.md)
- [Reconciliation and reporting](reconciliation-and-reporting.md)

## Sources

- [About Connect](https://docs.connect.worldline-solutions.com/getting-started/about-connect/) (docs)
- [Connect home](https://docs.connect.worldline-solutions.com/) (docs)
- [Connect API Reference — concepts / platforms](https://apireference.connect.worldline-solutions.com/s2sapi/v1/en_US/index.html) (docs; Ogone closed beta, Online Payment Acceptance)
- [Global Collect](https://worldline.com/en/home/main-navigation/solutions/merchants/global-collect) (marketing)
- [Go further in Europe](https://worldline.com/en/home/main-navigation/solutions/merchants/global-collect/go-further-in-europe-more-with-worldline) (marketing)
- [Operating models](https://docs.connect.worldline-solutions.com/getting-started/operating-models/) (docs; Full Service vs Gateway)
