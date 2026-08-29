# Troubleshooting

Lived incident first, then possible reasons, then how those would look. **No single root cause is established.**

Setup reference: [Setup webhooks](setup-webhooks.md). Canonical retry docs: [Webhook retry strategy](https://docs.bankingcircleconnect.com/docs/webhook-retry-strategy).

## What happened

Integration was almost wired.

1. Banking Circle reported the receiver returning **unauthorized (`401`)**. Cause was the **encryption token**. That was fixed.
2. **After the fix, no webhook POSTs were seen at the front door / ALB.** Subscription stayed **Active**, events still subscribed, Connect API auth fine (account balance works).
3. **Before** the fix, BC traffic **did** reach the endpoint (and was rejected on the bad token). So reachability used to work.
4. Working theory became **retry/backoff on the original failed notifications**. New payments were still going through, which raised the question: should those new events POST immediately, or sit behind the old retry clock?
5. Later hypothesis: the setup may actually be working, and the sandbox test endpoint (`POST .../clienttest/{subscriptionId}`) might **re-trigger the same notification already in backoff** rather than minting a new `eventId`. API **200 from `clienttest` means “events triggered”, not “the endpoint returned 2xx”.** Documented “send the failed one now” is portal **Restart delivery**.

## Possible reasons

Labelled **docs** (Connect docs / retry page) vs **inferred** (not stated, not contradicted).

### Ruled out for this incident

**Auto-deactivate after 11 retries (dedicated page) vs 10 (troubleshooting / Help Centre / `statusMessage`).** Docs: after the retry budget the subscription is deactivated, new events are dropped, pending failures are stored until reactivate. **Ruled out here: the subscription stayed Active.** Still an important trap for other incidents. The **10 vs 11** disagreement is an official-docs unknown; the dedicated retry page has an **11-row table**.

### Still in play

**Per-notification exponential backoff (15s … 48h).** **Docs.** Quiet ALB between retries is expected for the original `401`s. New events on an **Active** sub are only documented as dropped **after deactivate**. They should get a first-attempt POST. **Undocumented gap:** whether the subscription serializes all deliveries on one queue (new events waiting behind the old retry clock). **Inferred**, not stated.

**Returning `401` on decrypt failure counts as failed delivery.** **Docs** (any non-`2xx` retries). A still-wrong key still produces POSTs; BC encrypts with the configured key and POSTs ciphertext. **True silence at the ALB means BC is not sending, or packets are dropped in front of the app.** **Docs** for “wrong key ≠ TCP silence”; **inferred** which of those two silence modes this is.

**Key PUT can miss because `rowVersion` bumps on every retry.** **Docs.** Stale `If-Match` fails the update (webhook OpenAPI says `401` for a bad concurrency token; concurrency guide says `409` — conflict). Would still produce POSTs if the URL is unchanged; only explains a key that never actually rotated.

**`clienttest` may reuse the in-flight mock/notification** (same `eventId` / same failed delivery already in backoff) rather than minting a new one. **Inferred.** Docs say “a mocked payment matching your subscription will be sent, triggering a notification” and that the simulator “only triggers the webhook notification”. They do **not** say it creates a new `eventId`. They do **not** contradict reuse. **Restart delivery** = same failed notification, forced now, counter reset. **Docs.**

**URL / env mismatch, `octet-stream` WAF, sandbox vs prod IPs.** **Docs** that these fail delivery or drop traffic. Less likely as the *first* explanation because pre-fix `401`s prove BC reached the old URL from sandbox IPs. Becomes likely if the URL, WAF, or front door changed with the key fix. A JSON-only WAF dropping `application/octet-stream` would look like “nothing hit the app” while the ALB still saw POSTs.

**No events on a replacement subscription.** **Docs.** Create does not attach events. Duplicate URL is rejected.

**Client Services will not regenerate a webhook if the txn is already on the recon report.** **Secondary (KA-01119).** Backfill from the report / GET payments; do not wait for a resend of that txn.

## Possible explanations

How the remaining reasons would produce *this* timeline. These are hypotheses, not conclusions.

1. **Backoff only (original `401`s still retrying).** ALB is quiet between attempts (15s, 30s, 1m, 10m, 30m, 1h, 2h, 6h, 12h, 24h, 48h). Looks dead if you watch for a few minutes. **New payments should still POST immediately** unless there is an undocumented per-subscription send queue. If a real new payment after the fix is also silent, this explanation is insufficient.
2. **`clienttest` is not a new event.** Calling it returns API 200 (“triggered”) while the POST is the same in-flight failure, still on the backoff clock — or Restart-delivery semantics without resetting the counter. Would match “test looks fine on the API, still nothing new at the ALB”.
3. **The setup is working; we have not yet seen a first-attempt POST for a genuinely new `eventId`.** Discriminator below.
4. **Something in front of the app started dropping POSTs after the key fix** (WAF `octet-stream`, new URL, wrong env’s IP list). Pre-fix `401`s only prove the *old* path. Check ALB/WAF logs for the sandbox IP list and `Content-Type: application/octet-stream`, not only app logs.
5. **Key PUT never landed** (`rowVersion`). Endpoint would still get ciphertext POSTs encrypted with the *old* key. That is **not** ALB silence; it is POSTs that fail decrypt again. Does not explain a quiet front door unless combined with (1) or (4).

## Discriminator

A **real new payment after the key fix**, with a new `eventId`, on the Active subscription.

- If that POSTs: original silence was backoff / `clienttest` reuse / waiting on Restart delivery.
- If that is also silent: it is **not** backoff. Look at IP allowlist, `octet-stream` WAF, URL/env, and whether BC is sending at all.

Sandbox `clienttest` 200 is **not** that discriminator. Portal **Restart delivery** is the documented “send the failed one now”.

## Retry table (docs)

From [Webhook retry strategy](https://docs.bankingcircleconnect.com/docs/webhook-retry-strategy) (`updatedAt: 2025-11-18`). Intervals are **time after last retry**. Success = any `2xx`; then that notification cannot be resent.

| Retry | Interval after previous | Email |
|---|---|---|
| 1st | 00:00:15 | — |
| 2nd | 00:00:30 | — |
| 3rd | 00:01:00 | — |
| 4th | 00:10:00 | — |
| 5th | 00:30:00 | — |
| 6th | 01:00:00 | — |
| 7th | 02:00:00 | Warning (~3h 40min after first failure) |
| 8th | 06:00:00 | — |
| 9th | 12:00:00 | — |
| 10th | 24:00:00 | Warning (~45h 40min after first failure) |
| 11th | 48:00:00 | **Deactivation email** |

Cumulative to 11th retry ≈ **3d 21h 42m**. After the 11th unsuccessful retry: subscription deactivated; pending failures stored; **new events during deactivation do not notify**.

Other official pages still say **10** retries then deactivate (Add-event API, troubleshooting guide, `statusMessage`, Help Centre KA-01119). **Unknown which number is live.** Use the dedicated table as the detailed policy and do not pretend the conflict is resolved.

**Unknowns that matter here**

- Whether the failed-attempt counter is per notification or per subscription.
- Whether the subscription serializes deliveries (new events queued behind old retries).
- Whether `clienttest` mints a new `eventId` or re-fires the in-flight notification.
- Exact wire header names (`Checksum` vs `X-Bc-*`).
- 10 vs 11.

## Ack behaviour to change

Return **`2xx` first**, then decrypt. A decrypt failure that returns `401` is what BC counts as failed delivery. A wrong key still produces POSTs; silence is a different class of problem.
