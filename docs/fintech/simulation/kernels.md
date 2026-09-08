# Fintech-sim-lab — conceptual kernel catalogue

**Status:** **v1 accepted 2026-09-08** (educational; evolutionary — may add/merge kernels later)  
**Audience:** composable modular architecture for the simulation lab  
**Companion:** [`b2b-fintech-landscape-map.md`](../landscape/b2b-fintech-landscape-map.md) · study pack [`index.md`](../../study-material/index.md)  
**Snapshot:** 8 Sep 2026 (Europe/Dublin)  
**Quality bar:** concept-correct over complete. Kernels name reusable jobs, not vendor products. No proprietary Worldline / Adyen / B4B / Banking Circle field dictionaries.

---

## How to read this doc

| Term | Meaning in sim-lab |
| --- | --- |
| **Kernel** | Smallest conceptual unit with a clear contract: one job, owned state (or none), YAML-declared shape, explicit non-goals. |
| **YAML recipe** | Declares a kernel’s *contract/shape* (ports, policies, clocks, adapters)—not a dump of a vendor OpenAPI. |
| **Pod** | Composition of kernels that share an **in-pod bus** and a **bootstrap** order. A pod maps to a landscape *service type* (acquirer edge, EMI Oversight, bank rails, …). |
| **In-pod bus** | Intra-pod event/command channel. Cross-pod edges are explicit (HTTP, file, webhook)—never silent coupling. |
| **Bootstrap** | Ordered bring-up: secrets → clock → stores → ingress → workers. |

**Prove path (real stack → kernel graphs):**  
Worldline (acquirer) → ACI payment gateway → B4B (regulatory/TM; Oversight handoff at `B4BTMApproved`) → Banking Circle (rails, Connect webhooks, settlement).  
See §4. Landscape citations are conceptual (category numbers / section names), not API reverse-engineering.

---

## 1. Kernel design principles

What makes a good kernel:

1. **One job, one truth.** A kernel either *decides*, *holds state*, *transforms messages*, or *moves bytes*—not all four. If you need two truths (e.g. internal books vs bank evidence), that is two kernels.
2. **Contract over implementation.** YAML declares ports (commands/queries/events), idempotency keys, timeout budgets, and adapter kinds—not “call vendor X’s undocumented field.”
3. **Owned state is explicit—or explicitly none.** Stateful kernels name what they persist (ledger balances, party records, delivery attempts). Stateless kernels (codec, ingress adapter) must not grow a shadow database.
4. **Non-goals are load-bearing.** Every kernel lists what it must *not* do so pods do not silently absorb neighbouring jobs (e.g. webhook ACK ≠ book ledger).
5. **Composable, not decorative.** A kernel that only answers `/health` is not wired. Lab success = evidence flowing across the bus to a stored effect (see existing sim-lab “connected, not decorative”).
6. **Vendor-shaped, vendor-agnostic.** Shapes (idempotent create, pending→posted, signed webhook, allowlisted egress, cutoff calendar) transfer to prod conversation; payloads stay generic (`GB00SIM…`, fake HMAC secrets).
7. **Sharp over encyclopedic.** Prefer ~12–18 kernels that cover InfinitePay-class stitches; defer niche cells (Travel Rule CASP, BNPL, scheme arbitration portals) until a pod needs them.
8. **DES-friendly.** Time, cutoffs, retries, and auth timeouts are first-class (`clock-cutoff`); wall-clock sleeps are not substitutes for sim time.
9. **Control plane ≠ money plane.** Screening/TM/KYC gates may *block* handoff; only settlement/recon evidence (or an explicit settlement worker) may *book* customer/virtual ledgers after rail confirmation.
10. **Landscape-mapped.** Each kernel cites landscape categories that *use* it so pods can be assembled from the map’s collect → hold → move → recon stitch.

**Anti-patterns (reject in review):** god-orchestrator that owns ledger + webhooks + screening; “status API” kernel that also posts books; codec that embeds business approval; notifier that mutates balances on 2xx.

---

## 2. Kernel catalogue

### 2.1 Summary table

| id | name | one-line purpose | owns state? |
| --- | --- | --- | --- |
| `ledger` | Ledger | Double-entry pending/posted books of obligations | Yes — accounts, transfers |
| `account-directory` | Account directory | Resolve VIBAN/IBAN/routing aliases to licensed accounts | Yes — aliases, routing map |
| `payment-orchestrator` | Payment orchestrator | Saga/state machine across stages; does not hold money | Yes — payment case + stage |
| `auth-decision` | Auth decision | Card-style auth approve/decline/stand-in (ISO 8583-shaped) | None for holds — emits intents; **`ledger` pending is sole hold truth** |
| `capture-presentment` | Capture / presentment | Complete/capture after auth; presentment toward clearing | Soft — capture intents |
| `clearing-file-adapter` | Clearing file adapter | Ingest/emit batch clearing & report files | Cursor/watermark only |
| `settlement-window` | Settlement window | Net/gross settlement cycles & finality markers | Window & net positions |
| `webhook-notifier` | Webhook notifier | Listen queue → take → package → signed deliver → retry | Delivery attempts/subs |
| `http-ingress` | HTTP ingress | Authenticated HTTP edge; map request → bus command | None (or request log) |
| `scheme-message-codec` | Scheme message codec | Encode/decode rail/scheme message shapes | None |
| `kyc-kyb-gate` | KYC / KYB gate | Onboarding eligibility decision against party evidence | Decision + evidence refs |
| `sanctions-screener` | Sanctions / AML screener | Payment-time list / screening gate | Hits / case refs |
| `fraud-scorer` | Fraud scorer | Real-time abuse / ATO / mule risk score + policy action | Scores / rules version |
| `fx-rate-book` | FX rate book | Quote & lock FX for multi-currency legs | Rate snapshots / locks |
| `clock-cutoff` | Clock / cutoff | Sim time, calendars, cutoffs, timeout budgets | Sim clock & schedules |
| `identity-party-store` | Identity / party store | Canonical party, account holder, beneficiary records | Party graph |
| `secrets-policy` | Secrets / allowlist policy | Secret handles + egress allowlists (no raw prod secrets) | Policy config + handles |
| `audit-journal` | Audit journal | Append-only who/what/when for control & forensics | Immutable log |

**Count:** 18 kernels (upper end of the sharp band). Deferred candidates → §5.

---

### 2.2 Kernel details

#### `ledger`

- **Name:** Ledger  
- **Purpose:** Own the programmable books: pending holds and posted transfers that represent what the platform owes parties. Maps auth/hold → `pending`, clear/settle confirm → `post`, expiry/reversal → `void` (TigerBeetle / Modern Treasury–class two-phase mental model).  
- **Conceptual I/O:**  
  - *In:* `TransferIntent` (pending/post/void), balance queries, account open.  
  - *Out:* `TransferResult`, balance snapshots, imbalance/reject.  
- **State owned:** Chart of accounts, balances, pending transfers, posted journal (sim: memory OK; contract still immutable-shaped).  
- **Config surface (YAML):** currencies, account schemes, pending TTL, imbalance policy, **idempotency key space** (cross-cutting with `http-ingress` + `payment-orchestrator`—**not** a separate kernel).  
- **Non-goals:** Talking to rails; parsing clearing files; deciding KYC/fraud; emitting customer webhooks; being the bank’s books of record.  
- **Landscape:** Cat 13 (ledger / virtual accounts / treasury); also recon *consumer* of Cat 10 evidence via orchestrator—not a substitute for Cat 10.

#### `account-directory`

- **Name:** Account directory  
- **Purpose:** Map routing references (VIBAN/IBAN, virtual account aliases, BIC/sort hints) to the licensed omnibus/safeguarding account and internal ledger account ids.  
- **Conceptual I/O:** resolve / allocate / retire alias; lookup by external ref.  
- **State owned:** Alias → account bindings, allocation pools, lifecycle (active/frozen/retired).  
- **Config surface:** pools per currency/rail, format validators (fake `GB00SIM…` in lab), uniqueness rules.  
- **Non-goals:** Holding balances; initiating payments; issuing real BINs/PANs; FX.  
- **Landscape:** Cat 1, 2, 13 (wholesale rails / EMI / VIBAN); Cat 10 consumers match on these refs.

#### `payment-orchestrator`

- **Name:** Payment orchestrator  
- **Purpose:** Drive a payment *case* through stages (accept → screen → handoff → await rail → settle-signal → book instruction) without owning money or rail connectivity itself.  
- **Conceptual I/O:** create/cancel/advance case; subscribe to gate & rail events; emit stage commands to other kernels.  
- **State owned:** Case id, external_ref correlation, stage, correlation map (e.g. oversight id → rail payment id)—**not** balances.  
- **Config surface:** stage graph, timeout per stage, handoff predicates (e.g. terminal regulatory approve vs fail), **idempotency keys** (cross-cutting with `http-ingress` + `ledger`—**not** a separate kernel).  
- **Non-goals:** Screening logic; ledger posts; webhook crypto; encoding ISO messages.  
- **Landscape:** Cross-cutting stitch (collect → hold → move → recon); BaaS/EMI facades; payout orchestration (Cat 7, 9).

#### `auth-decision`

- **Name:** Auth decision  
- **Purpose:** Card-network-shaped authorization: approve, decline, partial, stand-in—under a latency budget. On approve it **emits a hold intent** to `ledger`; it does **not** own hold balances.  
- **Conceptual I/O:** auth request (amount, MERCH, PAN/token ref, cryptogram pass/fail flag); auth response + intent id for ledger pending.  
- **State owned:** Optional auth decision log / expiry timers only — **`ledger` pending is the sole hold truth** (owner-accepted v1).  
- **Config surface:** timeout budget, stand-in policy, partial-approve rules, link to `fraud-scorer` / `sca-challenge` (when composed).  
- **Non-goals:** Capture/clearing/settlement; full scheme dialect fidelity; inventing interchange tables.  
- **Landscape:** Cat 4 acquiring; Cat 5 issuing; Control & scheme — SCA/3DS path; study-lab Auth stage.

#### `capture-presentment`

- **Name:** Capture / presentment  
- **Purpose:** Turn an approved auth into a capturable financial presentment (under/over-capture, multi-completion, auth-expiry races)—feeding clearing.  
- **Conceptual I/O:** capture request against auth ref; presentment batch hints.  
- **State owned:** Capture intents vs auth remaining amount.  
- **Config surface:** over-capture policy, multi-capture allowed, link to auth expiry via `clock-cutoff`.  
- **Non-goals:** Scheme file formats (that is `clearing-file-adapter` + `scheme-message-codec`); net settlement.  
- **Landscape:** Cat 4; study-lab Capture stage; DMS vs SMS conceptual split (Lithic-class education).

#### `clearing-file-adapter`

- **Name:** Clearing file adapter  
- **Purpose:** Boundary for **batch files and standard reports**: pull/push cycles, parse into domain events, emit acknowledgements—without owning fee economics.  
- **Conceptual I/O:** file available / parse result / unmatched exception; watermark advance.  
- **State owned:** Ingest cursors, file checksums, exception quarantine refs.  
- **Config surface:** adapter kind (`card-clearing-report`, `ach-batch`, `wx-style-ops-report` *generic*), schedule via `clock-cutoff`, retry.  
- **Non-goals:** Authoritative Mastercard IPM / Visa BASE II field dictionaries (members-only); booking ledgers directly.  
- **Landscape:** Cat 4 recon reports; Cat 10; correspondent/clearing infrastructure (Cat 15) at file edge; Worldline-class SFTP report *shape* only.

#### `settlement-window`

- **Name:** Settlement window  
- **Purpose:** Model when value becomes final for a rail: cutoff calendars, net vs gross, multilateral netting positions, liquidity shortfall signals.  
- **Conceptual I/O:** open/close window; submit obligation; window-finalized event; shortfall alert.  
- **State owned:** Window instances, net positions, finality marks.  
- **Config surface:** calendar, timezone (lab: Europe/Dublin display; sim UTC OK), netting mode, rail profile (`sepa-inst`, `fps`, `scheme-net`, `rtgs`).  
- **Non-goals:** Customer webhook delivery; KYC; pretending EMI safeguarding equals FSCS.  
- **Landscape:** Cat 10, 15; A2A rails beyond OB; study-lab Settlement stage.

#### `webhook-notifier`

- **Name:** Webhook notifier  
- **Purpose:** Reliable egress notifications: **listen on a queue → take → package (sign) → distribute (HTTP POST) → retry / fail subscription**. Explicitly a first-class kernel (PSP Connect–shaped).  
- **Conceptual I/O:** enqueue delivery; delivery attempt result; subscription CRUD; DLQ/fail.  
- **State owned:** Subscriptions, attempt history, backoff cursor, failed-subscription flag.  
- **Config surface:** HMAC/signing profile, allowlist ref (`secrets-policy`), backoff schedule, max attempts, 2xx-only success.  
- **Non-goals:** Booking ledgers on ACK; creating rail payments; auto-subscribing sibling pods; being source of truth for settlement.  
- **Landscape:** Cat 10 (Connect / PSP status webhooks); Cat 1–2 partner callbacks. Lab: notifier service in fintech-sim-lab-src.

#### `http-ingress`

- **Name:** HTTP ingress  
- **Purpose:** Edge adapter: TLS termination shape, authN of callers, idempotency-key intake, map HTTP → bus commands/queries.  
- **Conceptual I/O:** HTTP request/response; emits domain commands.  
- **State owned:** None required (optional request/audit forward to `audit-journal`).  
- **Config surface:** routes → commands, auth mode (mTLS / HMAC / bearer-*fake*), body size, **idempotency header name** (cross-cutting with `ledger` + `payment-orchestrator`—**not** a separate kernel).  
- **Non-goals:** Business approval; holding balances; long retries (belong in workers/notifier).  
- **Landscape:** All API-facing cats (1–5, 7–10); ACI/gateway and Oversight *facade* shapes.

#### `scheme-message-codec`

- **Name:** Scheme message codec  
- **Purpose:** Pure transform between canonical domain payment messages and rail/scheme *shapes* (ISO 8583-ish auth, ISO 20022 pacs/camt *conceptual* envelopes, ACH batch rows)—no business decisions.  
- **Conceptual I/O:** encode(domain) / decode(bytes) → domain; validation errors.  
- **State owned:** None (schemas/version pins are config).  
- **Config surface:** dialect profile id, version, mandatory field set (lab-generic), unknown-field policy.  
- **Non-goals:** Approving payments; inventing proprietary vendor JSON as “the standard”; full members-only manuals.  
- **Landscape:** Cat 6 schemes; Cat 15 SWIFT/ISO; A2A rails; study materials ISO 20022 / ISO 8583 OSS hooks.

#### `kyc-kyb-gate`

- **Name:** KYC / KYB gate  
- **Purpose:** Onboarding-time eligibility: is this party allowed to hold/transact under the programme’s policy? Consumes evidence from `identity-party-store` + vendor-shaped check results.  
- **Conceptual I/O:** submit / decide / refresh; block or allow product access.  
- **State owned:** Gate decisions, evidence references, refresh due-at.  
- **Config surface:** policy packs (individual/company), refresh cadence, fail-closed vs review queue.  
- **Non-goals:** Payment-time sanctions (use `sanctions-screener`); fraud scoring; Travel Rule VASP messaging.  
- **Landscape:** Cat 11; EMI/BaaS onboarding (Cat 2, 7).

#### `sanctions-screener`

- **Name:** Sanctions / AML screener  
- **Purpose:** Payment-time (and periodic) list screening / AML pattern hooks that can block handoff—distinct from onboarding KYC and from pure card fraud.  
- **Conceptual I/O:** screen(party, payment); hit / clear / review; case open.  
- **State owned:** Hit records, case ids, list version pins.  
- **Config surface:** list sources (fake in lab), match thresholds, fail-closed, review SLA timers via `clock-cutoff`.  
- **Non-goals:** Replacing licensed TM vendors; filing real SARs; booking money.  
- **Landscape:** Cat 11–12; RegTech beyond KYC; B4B Oversight *regulatory/TM* conceptual slot before rail handoff.

#### `fraud-scorer`

- **Name:** Fraud scorer  
- **Purpose:** Score payment/auth/payout abuse (ATO, mule, card fraud) and emit allow / challenge / deny for orchestrator or `auth-decision`.  
- **Conceptual I/O:** score(features); action recommendation.  
- **State owned:** Optional feature cache / model version; not balances.  
- **Config surface:** model/policy version, challenge routing (to `sca-challenge` when present), shadow vs enforce.  
- **Non-goals:** Scheme chargeback ops; sanctions lists; KYC document OCR.  
- **Landscape:** Cat 12; acquirer-native fraud; issuing auth path.

#### `fx-rate-book`

- **Name:** FX rate book  
- **Purpose:** Provide quotable rates and short-lived locks for multi-currency payment legs and treasury sims.  
- **Conceptual I/O:** quote; lock; release/expire lock; convert amount.  
- **State owned:** Rate snapshots, locks, lock TTL.  
- **Config surface:** currency pairs, lock TTL, markup policy (lab-obvious), calendar.  
- **Non-goals:** Actually moving nostro liquidity; being a CASP; inventing real vendor spreads.  
- **Landscape:** Cat 8 FX / multi-currency; Cat 1 wholesale FX.

#### `clock-cutoff`

- **Name:** Clock / cutoff  
- **Purpose:** Discrete-event simulation time source: advance clock, fire cutoff calendars, auth/capture TTLs, webhook backoff schedules, settlement windows.  
- **Conceptual I/O:** now; schedule(event); advance(to); cancel.  
- **State owned:** Sim clock, scheduled events, calendar definitions.  
- **Config surface:** start time, calendars (TARGET, UK bank holidays *as data*), rail cutoff profiles.  
- **Non-goals:** Business decisions; hiding real `sleep` as “settlement.”  
- **Landscape:** Cross-cutting; Cat 10/15 timing; study-lab DES hook.

#### `identity-party-store`

- **Name:** Identity / party store  
- **Purpose:** Canonical store for people/companies/beneficiaries and their roles—inputs to KYC gates and payment parties.  
- **Conceptual I/O:** upsert party; link instrument/alias; query.  
- **State owned:** Party graph, roles, instrument links (token refs—not raw PAN).  
- **Config surface:** party schemas, uniqueness (email/reg no.), PII retention flags (lab: fake PII only).  
- **Non-goals:** Screening; ledger; vaulting PANs (`token-vault` deferred).  
- **Landscape:** Cat 11; Cat 2/7 programme parties; Cat 9 payee onboarding.

#### `secrets-policy`

- **Name:** Secrets / allowlist policy  
- **Purpose:** Hold *handles* to secrets and **egress allowlists** (webhook destinations, SFTP peers). Enforce “no arbitrary egress” in lab and prod-shaped configs.  
- **Conceptual I/O:** resolve secret handle; check allowlist(host); rotate handle.  
- **State owned:** Policy documents, allowlist entries, secret *references* (values from env/vault adapter—never commit real tokens).  
- **Config surface:** allowlist CIDRs/hostnames, HMAC secret handle ids, mTLS material refs.  
- **Non-goals:** Implementing a full HSM; WAF product; storing prod bank credentials in git.  
- **Landscape:** Cross-cutting security; Connect webhook setup hygiene; sim-lab CA/allowlist docs.

#### `audit-journal`

- **Name:** Audit journal  
- **Purpose:** Append-only record of control-plane and money-plane *decisions and effects* for forensics and sim assertions (“show me why this case blocked”).  
- **Conceptual I/O:** append(event); query by correlation id.  
- **State owned:** Immutable log segments.  
- **Config surface:** retention, redaction policy, required correlation fields.  
- **Non-goals:** Being the ledger; replacing recon reports; mutable “admin edit.”  
- **Landscape:** Cross-cutting; supports Cat 10/12/11 accountability narratives.

---

## 3. Common pod recipes (landscape service types)

Pods = kernels + in-pod bus + bootstrap. Names are educational, not products.

### 3.1 Recipe index

| Pod recipe | Landscape service type | Primary kernels | Typical edges out |
| --- | --- | --- | --- |
| `pod-acquirer-edge` | Cat 4 acquiring / gateway | `http-ingress`, `payment-orchestrator`, `auth-decision`, `capture-presentment`, `fraud-scorer`, `ledger` (merchant float *sim*), `clearing-file-adapter`, `clock-cutoff`, `secrets-policy`, `audit-journal` | Settlement reports → platform; optional `webhook-notifier` to merchant |
| `pod-gateway-facade` | Gateway-only (funds elsewhere) | `http-ingress`, `payment-orchestrator`, `scheme-message-codec`, `fraud-scorer`, `clock-cutoff`, `audit-journal` | Forwards to acquirer pods; **no** funds ledger of record |
| `pod-emi-oversight` | Cat 2 EMI regulatory / pre-settlement | `http-ingress`, `payment-orchestrator`, `kyc-kyb-gate`, `sanctions-screener`, `fraud-scorer`, `identity-party-store`, `webhook-notifier` (client callbacks), `secrets-policy`, `audit-journal`, `clock-cutoff` | Handoff command → bank-rails pod on terminal approve |
| `pod-bank-rails` | Cat 1 wholesale rails + Cat 10 Connect | `http-ingress`, `payment-orchestrator`, `account-directory`, `ledger` (bank books *sim*), `scheme-message-codec`, `settlement-window`, `webhook-notifier`, `fx-rate-book`, `clearing-file-adapter`, `clock-cutoff`, `secrets-policy`, `audit-journal` | Webhooks + **GET status / recon pull** evidence ports; clearing participation |
| `pod-ledger-recon` | Cat 13 + Cat 10 matching | `ledger`, `account-directory`, `payment-orchestrator` (match jobs), `clearing-file-adapter`, `webhook-notifier` (ingest side optional), `clock-cutoff`, `audit-journal` | **Only** money-of-record for customer/virtual books; evidence = webhook hints + GET/recon pulls; book on Processed/recon |
| `pod-open-banking-tpp` | Cat 3 AISP/PISP | `http-ingress`, `payment-orchestrator`, `identity-party-store`, `kyc-kyb-gate` (consent), `webhook-notifier`, `secrets-policy`, `audit-journal` | ASPSP/rail—not a float holder unless EMI permission composed |
| `pod-issuer-processor` | Cat 5 issuing | `http-ingress`, `auth-decision`, `ledger`, `fraud-scorer`, `scheme-message-codec`, `clock-cutoff`, `audit-journal` (+ deferred `token-vault`, `sca-challenge`) | Scheme/auth network sim |
| `pod-fx-treasury` | Cat 8 | `fx-rate-book`, `ledger`, `payment-orchestrator`, `account-directory`, `settlement-window`, `clock-cutoff` | Rails pod for payout legs |
| `pod-payout-ops` | Cat 9 disbursements | `http-ingress`, `payment-orchestrator`, `identity-party-store`, `sanctions-screener`, `ledger`, `webhook-notifier`, `clock-cutoff` | May batch later (`payout-batcher` deferred) |
| `pod-risk-control` | Cat 11–12 side car | `kyc-kyb-gate`, `sanctions-screener`, `fraud-scorer`, `identity-party-store`, `audit-journal`, `clock-cutoff` | Decision events into orchestrators |

### 3.2 Bootstrap pattern (all pods)

```
secrets-policy → clock-cutoff → stores (identity/ledger/directory)
  → codecs/adapters → decision gates → orchestrator → http-ingress
  → workers (notifier, settlement, file pollers)
```

### 3.3 In-pod bus event grammar (conceptual)

- `Command.*` — imperative (CreatePayment, ScreenPayment, PostTransfer)  
- `Fact.*` — immutable outcomes (ScreenCleared, AuthApproved, WindowFinalized, DeliverySucceeded)  
- `Signal.*` — hints (WebhookReceived)—**never** sole booking authority  

---

## 4. InfinitePay-path pod dissection

Real stack under study: **Worldline (acquirer) → ACI payment gateway → B4B (regulatory/TM, handoff at `B4BTMApproved`) → Banking Circle (rails, Connect webhooks, settlement)**.

Kernel graphs below are **conceptual**. Status names that appear in partner docs (`B4BTMApproved`, BC `Processed` / `Booked`) are cited as *handoff vocabulary* already used in the study notes—not as a field dictionary.

### 4.1 End-to-end swimlane

```
Buyer
  │ card / A2A
  ▼
[pod-acquirer-edge ≈ Worldline]          collect / auth / capture / remittance evidence
  │ settlement / reports (files)
  ▼
[pod-gateway-facade ≈ ACI]               technical gateway; orchestration toward programme
  │ payment create (platform → EMI)
  ▼
[pod-emi-oversight ≈ B4B Oversight]      KYC/TM/sanctions path; client webhook until handoff
  │ terminal: Approved → handoff    OR   Failed → stop (no rail payment)
  ▼
[pod-bank-rails ≈ Banking Circle]        VIBAN/rails/FX; Connect notifier; GET/recon truth
  │ Processed / Booked signals (order ≠ guaranteed)
  ▼
[pod-ledger-recon ≈ platform books]      settlement worker books virtual ledger; recon drift control
```

**Hard rules (from connected-path doctrine):**  
- Do **not** book customer/virtual ledger on Oversight approval alone.  
- Oversight webhook **stops at handoff**; settlement truth is bank GET/recon (webhooks = hints).  
- Connect subscriptions are **integrator-owned** (not auto-created by Oversight).  
- Webhook 2xx ≠ book.  
- **Customer/virtual books live only in `pod-ledger-recon`** — `pod-emi-oversight` is never money-of-record (owner-accepted v1).  
- **Book on rail `Processed` (or recon/GET evidence)**; `Booked` is unordered recon-only (owner-accepted v1).  
- **Idempotency** is a cross-cutting contract on `http-ingress` + `ledger` + `payment-orchestrator` — not a kernel.

### 4.2 Worldline ≈ `pod-acquirer-edge` graph

```
http-ingress ──► payment-orchestrator ──► auth-decision ──► ledger (pending hold)
                         │                      │
                         │                      └► fraud-scorer
                         ├► capture-presentment ──► (presentment facts)
                         └► clearing-file-adapter ◄── clock-cutoff (report cycles)
secrets-policy (SFTP/report peers, TLS)
audit-journal (auth/capture/report facts)
```

**Money plane:** acquirer collects; remittance/settlement evidence leaves as **files/reports** (generic ops/financial report *shape*). Full-service vs gateway-only is a **pod config flag** (funds-in-pod vs funds-external)—not a new kernel.

### 4.3 ACI ≈ `pod-gateway-facade` graph

```
http-ingress ──► payment-orchestrator ──► scheme-message-codec
                         │
                         ├► fraud-scorer (optional edge checks)
                         └► commands toward programme / EMI ingress
clock-cutoff · secrets-policy · audit-journal
```

**Non-goal of this pod:** being EMI safeguarding or bank settlement. It is the **technical composition point** (gateway) between acquirer evidence and programme APIs.

**InfinitePay default:** keep **`pod-gateway-facade` separate** (ACI-shaped) from `pod-acquirer-edge` (Worldline-shaped). Config-collapse into one pod is allowed later if a lab does not need the split—not the default for this office stack.

### 4.4 B4B Oversight ≈ `pod-emi-oversight` graph

```
http-ingress ──► payment-orchestrator
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
 identity-party-store  kyc-kyb-gate  sanctions-screener
         │               │               │
         └───────────────┴► fraud-scorer ┘
                         │
                         ├─ Fact.TmApproved ──► handoff port → bank-rails
                         ├─ Fact.TmFailed   ──► terminal stop
                         └─ webhook-notifier ──► client callback_url (pre-settlement statuses only)
secrets-policy · clock-cutoff · audit-journal
```

**Handoff vocabulary (conceptual):** terminal regulatory approve carries a **rail payment correlation id** into the bank-rails pod; fail does not create a rail payment. No ledger booking of customer available balance here.

**YAML recipe toggles (Fintech default):**  
- `client_callback_pre_settlement: true` — documented Oversight statuses through handoff.  
- `client_callback_post_handoff_opportunistic: false` (default) — optional sim of undocumented same-`callback_url` fire when BC processes; **non-booking noise**; settlement truth remains bank GET/recon/webhooks.  
- Keep **`sanctions-screener` ⊥ `fraud-scorer` ⊥ `kyc-kyb-gate`** as separate kernels (do not collapse to a single `risk-gate` for InfinitePay realism).

### 4.5 Banking Circle ≈ `pod-bank-rails` graph

```
http-ingress ──► payment-orchestrator ──► account-directory
                         │                      │
                         ├► scheme-message-codec ┤
                         ├► fx-rate-book         │
                         ├► ledger (bank books) ◄┘
                         ├► settlement-window ◄── clock-cutoff
                         └► webhook-notifier ──► integrator URLs (allowlisted)
                                  │
                         clearing-file-adapter / recon GET ports (evidence)
secrets-policy · audit-journal
```

**Notifier kernel path (explicit):** queue listen → take → package (sign) → POST → 2xx done / else retry → exhaust → fail subscription.  
**Status semantics (conceptual):** rail “sent/processed” vs “booked on account” may be **unordered**; absence of a single “Settled” umbrella status is a modelling feature, not a bug.

**First-class evidence ports (not only batch files):**  
- **Pull status** — GET payment/status by rail payment id (YAML profile on `http-ingress` query routes + `payment-orchestrator` poll recipe).  
- **Recon ingest** — intraday/end-of-day recon report pulls (YAML profile on `clearing-file-adapter`, e.g. `bc-intraday-recon`).  
- Webhook deliveries = **hints**; GET/recon lines = **evidence** for `pod-ledger-recon`.  
- Promote deferred `status-poller` only if these YAML profiles prove too awkward for the InfinitePay midday sweeper.

### 4.6 Platform recon ≈ `pod-ledger-recon` graph

```
Signals: Connect deliveries (hint) + GET/recon lines (evidence)
                │
                ▼
payment-orchestrator (settlement worker) ──► ledger (post/void)
                │
                ├► account-directory (match VIBAN/refs)
                └► audit-journal
clock-cutoff (sweep schedules)
```

**Only this worker** (or an equally explicit booking policy) posts the **customer/virtual** ledger after rail confirmation.

**Evidence ports (Fintech default):** accept Connect webhook hints **and** pull-status / recon lines from `pod-bank-rails`. Booking predicate: **`Processed` or matching recon/GET evidence** — not Oversight approve, not webhook 2xx alone, not unordered `Booked` as the sole gate. Sweep schedules live on `clock-cutoff` (e.g. midday recon + leftovers GET).

### 4.7 Correlation spine (lab success)

One `external_ref` must trace:

`acquirer/gateway payment refs → EMI case id → rail payment id → notifier event id(s) → ledger transfer id(s)`

Isolated health pings or mock replays without this spine are **not** a connected InfinitePay path.

---

## 5. Out of scope / deferred kernels

Worth naming so they are not smuggled into the 18:

| Deferred id | Why deferred | When to promote |
| --- | --- | --- |
| `payout-batcher` | Mass disbursement scheduling is a recipe on `payment-orchestrator` + `ledger` until volume forces a kernel | Marketplace payout pods at scale |
| `sca-challenge` | 3DS/SCA challenge orchestration; needed for CNP liability-shift sims | Card-issuing / EU CNP acquirer pods |
| `token-vault` | PAN/network-token vault (PCI); distinct from `identity-party-store` | Issuer processor / network-token labs |
| `dispute-case` | Chargeback / RDR–Ethoca-class casework | Acquirer dispute ops sims |
| `travel-rule-exchange` | CASP originator/beneficiary messaging | Stablecoin / BC CASP corridors |
| `stablecoin-bridge` | Fiat↔stablecoin settlement adapter | Cat 14 labs |
| `consent-grant-store` | OB consent objects | Deep Cat 3 TPP pods |
| `mandate-store` | SDD mandates | Direct-debit pods |
| `liquidity-position` | Nostro/RTGS intraday liquidity beyond `settlement-window` | Wholesale RTGS stress sims |
| `scheme-fee-engine` | Interchange/MDR calculators | Economics labs—**no invented fee tables** |
| `safeguarding-segregation` | Explicit EMI safeguarding accounting | Regulatory stress scenarios |
| `app-fraud-recall` | A2A recall / APP-fraud regimes | UK FPS-focused pods |
| `status-poller` | Dedicated poll worker for GET payment/status + recon report pulls | Only if YAML profiles on `clearing-file-adapter` / orchestrator prove too awkward for InfinitePay sweeper |

Also **out of deep scope** with the landscape map: bancassurance, wealth/robo, BNPL credit decisioning.

---

## 6. Fintech domain decisions (owner-accepted / v1-accepted 2026-09-08)

Accepted as **v1 defaults** (may evolve later):

| # | Topic | Accepted default |
| --- | --- | --- |
| D1 | Customer / virtual books | **Only** `pod-ledger-recon` — Oversight never money-of-record; no book on TM approve |
| D2 | Sanctions / fraud / KYC | Keep **`sanctions-screener` ⊥ `fraud-scorer` ⊥ `kyc-kyb-gate`** (no single `risk-gate` for InfinitePay) |
| D3 | ACI vs Worldline | Keep separate **`pod-gateway-facade`** (ACI); config-collapse later optional |
| D4 | Processed vs Booked | Book customer ledger on **`Processed` or recon/GET evidence**; **`Booked` = unordered recon-only** |
| D5 | Idempotency | Cross-cutting on `http-ingress` + `ledger` + `payment-orchestrator` — **no** idempotency kernel |
| D6 | Post-handoff Oversight callback | Optional YAML toggle; **non-booking** noise; settlement truth stays bank evidence |
| D7 | Auth hold ownership | **`ledger` pending is the sole hold truth**; `auth-decision` emits intents only |

## 7. Open questions still for the owner

1. **Kernel vs adapter boundary for Worldline reports:** Keep `clearing-file-adapter` generic with YAML “report profiles,” or split `report-ingress` (SFTP) from `clearing-codec`?  
2. ~~EMI customer books location~~ → **locked as D1** above (confirm or override).  
3. ~~TM fusion~~ → **locked as D2** above (confirm or override).  
4. ~~Auth hold ownership~~ → **closed:** `ledger` **pending is the sole hold truth**; `auth-decision` emits intents only (one-truth rule).  
5. ~~ACI pod necessity~~ → **locked as D3** above (confirm or override).  
6. **Bus technology:** In-process channels for unit DES vs NATS/Redpanda for multi-container pods—same kernel contracts?  
7. **YAML recipe schema versioning:** One `kernel.v1` schema with `kind:` per id, or per-kernel CRD-like files?  
8. **Failure injection:** First-class `fault-injector` kernel vs test harness outside catalogue?  
9. ~~BC unordered Processed/Booked~~ → **locked as D4** above (confirm or override).  
10. **Promotion path:** Which deferred kernel should be first after InfinitePay happy path—`sca-challenge`, `dispute-case`, or `payout-batcher`?

---

## Appendix A — Candidate merge/split log

| Candidate (input list) | Decision |
| --- | --- |
| ledger (pending/posted) | **Kept** `ledger` |
| account-directory (VIBAN/IBAN) | **Kept** |
| payment-orchestrator | **Kept** |
| auth-decision | **Kept** |
| capture/presentment | **Kept** `capture-presentment` |
| clearing-file-adapter | **Kept** |
| settlement-window | **Kept** |
| webhook-notifier | **Kept** (explicit) |
| http-ingress | **Kept** |
| scheme-message-codec | **Kept** |
| kyc/kyb gate | **Kept** `kyc-kyb-gate` |
| sanctions/aml-screener | **Kept** `sanctions-screener` (AML patterns may share pod with fraud) |
| fraud-scorer | **Kept** |
| 3ds/sca-challenge | **Deferred** `sca-challenge` |
| token-vault | **Deferred** |
| dispute/chargeback-case | **Deferred** `dispute-case` |
| fx-rate-book | **Kept** |
| payout-batcher | **Deferred** (recipe first) |
| clock/cutoff (DES) | **Kept** `clock-cutoff` |
| identity/party-store | **Kept** |
| secrets/allowlist policy | **Kept** `secrets-policy` |
| audit-journal | **Kept** |

## Appendix B — Landscape citation map (kernels → categories)

| Kernel | Primary landscape anchors |
| --- | --- |
| `ledger`, `account-directory` | Cat 13; contrast Cat 10 |
| `payment-orchestrator` | Stitch § “How an integrator typically stitches”; Cat 7/9 |
| `auth-decision`, `capture-presentment` | Cat 4/5; Control SCA; study Auth/Capture |
| `clearing-file-adapter`, `settlement-window` | Cat 10/15; A2A rails; study Clearing/Settlement |
| `webhook-notifier` | Cat 10 Connect/status; EMI callbacks |
| `http-ingress`, `scheme-message-codec` | API edges; Cat 6/15 message plumbing |
| `kyc-kyb-gate`, `identity-party-store` | Cat 11 |
| `sanctions-screener`, `fraud-scorer` | Cat 11–12; RegTech beyond KYC; Oversight TM slot |
| `fx-rate-book` | Cat 8 |
| `clock-cutoff`, `secrets-policy`, `audit-journal` | Cross-cutting / often-missed integrator layers |

---

*Educational architecture for fintech-sim-lab. Not legal advice; not a vendor integration guide. Re-check registers and contracted APIs before production use.*
