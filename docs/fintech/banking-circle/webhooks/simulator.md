# Webhook simulator

Contract for a **fake Connect notifier**, not a clone of Banking Circle’s protocol. A lab succeeds when a payment in the fake bank path causes a **decryptable POST** on the receiver, and a **failed** receiver still **retries**.

This is a simulation of Connect behaviour already filed in Avram. Do not treat the simulator’s wire format as their production spec.

Sources: [Setup webhooks](setup-webhooks.md) · [Troubleshooting](troubleshooting.md) · [FAQ](faq.md) · [Payment confirmation](../payment-confirmation.md) · [Retry strategy](https://docs.bankingcircleconnect.com/docs/webhook-retry-strategy)

## Must implement

| Behaviour | Simulator rule |
|---|---|
| Subscription ≠ events | Two steps: create subscription, then add events. Unique URL. Status Inactive / Active. Activate / deactivate. |
| Inbound POST | Binary **AES-256-GCM**, 32-char key. Checksum / nonce / auth tag present. Receiver **ACK `2xx` within 10s BEFORE decrypt**. Returning `401` on decrypt = **failed delivery**. |
| Retry | Per **notification**, not a global pause on an Active sub. Any non-`2xx` retries. Pending stored. **New events while Inactive are dropped.** `rowVersion` bumps **every retry**. |
| Backoff | 15s, 30s, 1m, 10m, 30m, 1h, 2h, 6h, 12h, 24h, 48h, then **deactivate**. Official pages disagree **10 vs 11**; implement the **11-row** table and note the conflict. |
| Delivery kinds | Explicit modes so labs do not confuse them: **`new_event`**, **`retry_failed`**, **`replay_mock`**. Sandbox `clienttest` *may* be the same in-flight notification (undocumented) — do not hide that behind a single “ping”. |
| IP allowlist | **Enforced** (compose / nginx), not a comment. Real Connect has distinct sandbox vs prod lists. Sim uses **local CIDRs** and documents how to swap to the live lists. |
| TLS | TLS 1.2 required. Optional **mTLS stub** (off by default). |
| Events (v1) | `OutgoingPaymentProcessed`, `IncomingPaymentProcessed`, `MissingFunding`, `Rejected`, `Booked`, `PaymentStatus` (**`payload` not `payment`**). Idempotency on `eventId`. `Processed` vs `Booked` **unordered**. |

Active subscription: **do not** block new events because an older notification is retrying. Docs never say a global send queue.

## Stubbed / not Connect

- No portal UI, no deactivation email, no 50MB payload, no live Azure IP ranges in default config.
- Header **wire names** (`Checksum` vs `X-Bc-*`) and key encoding (raw 32 chars vs Base64) stay **configurable**; official samples disagree. Pick one default, document it.
- Not a byte-for-byte protocol clone. Enough for a receiver to decrypt, ACK, fail, and retry.

## Lab success

1. Create Active subscription + events, unique URL, local allowlist.
2. `new_event` for an outgoing processed payment → receiver gets ciphertext POST, decrypts, `2xx`.
3. Break the receiver (non-`2xx` or `401` on decrypt) → **`retry_failed`** follows the 11-row table; `rowVersion` moves; after the last miss the sub deactivates; **new** events during Inactive are dropped.
