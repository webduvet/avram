# Testbed vision and kernels primer

Session notes for the fintech-sim-lab **product thesis** and a layman + technical primer on **kernels / pods / recipes**.

**Snapshot:** 2026-09-10 (Europe/Dublin)

**Links:** [Kernel catalogue](kernels.md) · [Landscape map](../landscape/b2b-fintech-landscape-map.md) · [Connected path](connected-path.md)

**Lab repo:** [webduvet/fintech-sim-lab](https://github.com/webduvet/fintech-sim-lab) — open review [PR #1](https://github.com/webduvet/fintech-sim-lab/pull/1) (kernels/pods review; recipes include `recipes/ledger-recon`).

---

## 1. Product thesis / future plans

### North star

Replace standalone `cmd/*` mocks with **assembled pods** after harness parity per hop (dual stack is transitional only).

### Product

A fintech integration/regression **testbed** — not fake vendor products.

**Motivation:** InfinitePay-class work often falls back to ad-hoc in-house mocks. Role fog (acquirer vs PayFac) is common. Generic working services help design against the real stitch.

Also: **education** and **follow-the-money**.

### Console vision (2026-09-10)

- System map with pipes.
- Trigger money (e.g. acquirer settlement file + lump sum to a safeguarding account).
- Animate **green** (money) / **red** (regulatory checks).
- Click a line to stop/break flow.
- Monitor each box’s inputs and outputs.
- Fabulous animation.
- Built on **pods + correlation spine**.

### Success metric idea

One `external_ref` lighting every hop; deliberate break at one pipe with a clear failing assertion.

### Polish priorities

1. Harness on pods.
2. Retire standalones after parity.
3. Avoid a dual-stack museum.
4. Docs that teach the map.

---

## 2. Combined primer — kernels / pods / recipes

### Layman

| Term | Meaning |
| --- | --- |
| **Kernel** | One financial job (book, screen, webhook, VIBAN map, status walk) — **not** a company. |
| **Pod** | Kernels snapped together behind one door (one HTTP address) ≈ a mocked service. |
| **Recipe YAML** | The **only** place vendor names / status strings live. Go kernels stay brandless. |

### Technical

- **Pod** = `cmd/pod` + recipe dir:
  - `pod.yaml` — kernels list + wiring
  - `kernels/*.yaml` — kind + id + spec instances
- **Naming:**
  - pod = compose(kernel_A@spec₁, …) + wiring
  - each `kernels/*.yaml` = one configured instance
  - recipe = the whole folder
- **Wiring** routes **topics only** (A→B, optional field guard). It never rewrites money fields.
- **Money / control (structural):**
  - ledger present ⇒ money plane
  - else control-only (cannot be money-of-record)
- **Bus message kinds:**
  - **Command** — directed
  - **Fact** — broadcast
  - **Signal** — external hint; never sole booking authority
  - Recipes **cannot** emit Signals
- **Publish is synchronous:** returns after immediate subscribers react → HTTP ingress can answer from Facts in the same wave. `hop_delay` stages are clock-async.
- **Cross-pod** = HTTP / files / webhooks only.

---

## 3. FAQ

**Q: Ultimate goal — replace standalones with pods?**  
**A:** Yes. Cutover only after harness parity per hop.

**Q: Elaborate pod / wiring and money / control?**  
**A:** See the primer above.

**Q: Elaborate synchronous delivery?**  
**A:** See the primer. Note: EvidenceAccepted then Processed on the clock.

**Q: Recipe ledger — is a ledger alone just a book?**  
**A:** `kind:ledger` = book only (pend / post / void, conservation). `recipes/ledger-recon` = platform money-of-record pod using a ledger instance plus settlement / hints / aliases / reports / api; only Processed wires PostTransfer. Bank-rails / acquirer pods have their own ledger instances for bank / acquirer books.

**Q: Is ledger essentially a DB with account / transaction / ledger_entry?**  
**A:** Conceptually a money store. Today: in-memory maps, not a durable 3-table schema. A thin conservation ledger is enough for cross-system lifecycle + regulatory sims until queryable history is needed. An audit-grade DB is not required for the current thesis.

---

## Related

- [Kernel catalogue](kernels.md)
- [B2B fintech landscape map](../landscape/b2b-fintech-landscape-map.md)
- [Connected path](connected-path.md)
- [fintech-sim-lab PR #1](https://github.com/webduvet/fintech-sim-lab/pull/1) · `recipes/ledger-recon`
