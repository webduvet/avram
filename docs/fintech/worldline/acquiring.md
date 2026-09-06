# Worldline as acquirer

How Worldline sits in the funds path, versus gateway-only processing, and how Global Collect markets local / multi-acquirer routing. Operational settlement recon for this stack is on [Reconciliation and reporting](reconciliation-and-reporting.md) (SFTP pull of standard reports).

Parent: [Worldline](index.md)

## Payment models (funds vs gateway)

Connect FAQ on reconciliation and reporting distinguishes models by **who handles funds**:

| Model | Funds | Highest transaction statuses (FAQ) | Reconciliation |
|---|---|---|---|
| Worldline handles funds (non-gateway) | Worldline collection / remittance applies | Beyond gateway caps (full lifecycle through capture / remittance as configured) | Between merchant and Worldline; standard WX + Financial Report |
| **Gateway** payment model | Collection and remittance are between the **merchant and the acquiring banks**; Worldline does **not** handle funds | **Pending Approval** (gateway accounts) / **Capture Requested** (gateway payment model) | Between the **acquirer, merchant, and Worldline** |

Source (docs): [Reconciliation and reporting FAQ](https://docs.connect.worldline-solutions.com/support/faq/connect/reconciliation-and-reporting) — including “Why can't I see the collection reports on my gateway payment model account?”

Collection reports are **decommissioned** on the GlobalCollect platform regardless; standard reports are WX (Operational) and the Financial Report. See [Reconciliation and reporting](reconciliation-and-reporting.md).

## Global Collect — local acquiring / multi-acquirer routing

**Product marketing** (not Connect file/API contracts):

- Global Collect is pitched for cross-border ecommerce with **direct connections to local acquirers**, multi-acquirer connectivity, and **smart / AI-powered routing** aimed at higher authorisation rates and lower cost (fallback across acquirers; route by currency, card/scheme, region, fee).
- Europe-focused marketing states Worldline owns acceptance, acquiring, and issuing; Global Collect offers domestic and cross-border processing in Europe and **local acquiring across the EEA** with smart routing.

Sources (marketing):

- [Global Collect](https://worldline.com/en/home/main-navigation/solutions/merchants/global-collect)
- [Go further in Europe](https://worldline.com/en/home/main-navigation/solutions/merchants/global-collect/go-further-in-europe-more-with-worldline)

Do not treat marketing claims as substitute for MID/contract setup or for report field specs.

## In-stack operational path

Settlement / operational recon used here is **pull of standard reports over Managed File Transfer (SFT / SFTP)** — primarily WX and Financial Report — not reliance on gateway-only console downloads. Details, cadence, and Documented vs Observed delivery notes: [Reconciliation and reporting](reconciliation-and-reporting.md).

## Sources

- [Reconciliation and reporting FAQ](https://docs.connect.worldline-solutions.com/support/faq/connect/reconciliation-and-reporting) (docs)
- [Reporting overview](https://docs.connect.worldline-solutions.com/reporting/) (docs)
- [Global Collect](https://worldline.com/en/home/main-navigation/solutions/merchants/global-collect) (marketing)
- [Go further in Europe](https://worldline.com/en/home/main-navigation/solutions/merchants/global-collect/go-further-in-europe-more-with-worldline) (marketing)
