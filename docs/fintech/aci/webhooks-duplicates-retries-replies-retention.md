# ACI webhooks: duplicates, retries, replies, retention

How ACI OPP webhooks behave for deduplication, retries, endpoint pausing, reply codes and retention.
Back to [ACI](index.md).

**Labels:** **Documented** = ACI primary docs (source tag + short verbatim quote). **Inferred** = integrator reasoning, not in ACI docs. **Not documented** = searched for, not found in the sources below.

**Verification:** all three sources re-fetched live on 2026-10-27; every quote below checked against the live text.

## Sources

| Tag | Page | "Last updated" shown (checked 2026-10-27) |
| --- | --- | --- |
| [W] | [Webhooks guide](https://docs.aciworldwide.com/tutorials/webhooks) | May 12, 2026 |
| [F] | [Webhooks FAQ](https://docs.aciworldwide.com/support/webhooks) | May 12, 2026 |
| [R] | [Data Retention Policy](https://docs.aciworldwide.com/reference/data-retention-policy) | December 2, 2025 |

## 1. Duplicate key

- **Documented [W]:** decrypted body is `{type, action, payload}`. `type` is "PAYMENT, REGISTRATION, SCHEDULE, or RISK". `action` is "Only present for REGISTRATION events. Indicates the change: CREATED, UPDATED, or DELETED". `payload` "mirrors the response you would receive from the corresponding API".
- **Documented [W]:** "Multiple final messages: You may receive more than one final status (e.g., success + failure). Deduplicate based on transaction ID and status."
- **Documented [W]:** "No guaranteed order: Events may arrive out of sequence."
- **Documented [W]:** "The same notification is sent to all active webhook URLs configured at or above the entity level".
- **Documented [F]:** "If the notification is for a transaction status change, you will get at least one of the following data points: id, referenceId, registrationId, or merchantTransactionId."
- **Documented [W]:** IV "Sent in HTTP header X-Initialization-Vector (hexadecimal)"; tag "Sent in HTTP header X-Authentication-Tag (hexadecimal)". Wrapper None: "Content-Type: text/plain" (raw hex body); JSON wrapper: "Content-Type: application/json", body `{"encryptedBody": "hexadecimal_string"}`.
- **Not documented:** any notification id or delivery id, in body or headers; any headers beyond the two above.
- **Inferred:** `payload.id` is the transaction/registration id, not a notification id, so two notifications can share `type` + `payload.id` (success then failure; registration CREATED then UPDATED). Capture/refund appear to be separate transactions pointing back (the [W] RISK example has `referencedId`). Better key: `type` + `payload.id` + `result.code` (+ `action`) + timestamp, or a hash of the decrypted body. A current-state view must not just keep the last arrival, because delivery is out of order.

## 2. Retries

- **Documented [W]:** "No response within 30 seconds → marked as failed"; "Non-2xx HTTP response → marked as failed".
- **Documented [W]:** "1 min → 2 min → 4 min → 8 min → 15 min → 30 min → 1 hour → daily (up to 30 days)".
- **Documented [F]:** "the retrial process will not exceed more than 30 days".
- **Inferred (calculated):** attempt count is not stated; from the intervals, about 7 retries in the first ~2 hours, then daily.
- **Not documented:** whether a retry is byte-identical (same IV/tag) or re-encrypted. **Inferred:** dedupe on decrypted content, not raw bytes.

## 3. Pausing / disabling

- **Documented [W]:** "Retries pause if all messages fail at a given interval; resume once delivery succeeds".
- **Documented [W]:** "Email with up to 100 failed notifications per endpoint" (daily failure summary).
- **Documented [W]:** "Failed messages stored for 30 days, then purged".
- **Documented [F]:** "Failing to take any action, leads to losing all the notifications beyond the retry period."
- **Not documented:** automatic disabling of the endpoint, or any failure-count threshold.

## 4. Reply codes

- **Documented [W]:** "it must respond with a 2xx HTTP status code (e.g., 200 OK). This confirms successful delivery and prevents retries." A "non-2xx status code" means the webhook "is considered undelivered and will be retried".
- **Inferred:** no exception is documented for 4xx, so 400/401 are retried like any failure (daily, up to 30 days).
- **Documented [W]:** only advice on bad payloads: "Handle decryption errors gracefully to avoid processing invalid payloads."
- **Inferred:** reply 200 for a permanently malformed body (after logging/alerting); keep 401 or 5xx for decryption/key failures so retries can recover the messages once fixed. 200 for a duplicate and 5xx when storage/queue is down are consistent with the docs.
- **Documented [W]:** "Click to Test" "sends a dummy notification to your configured URL"; the test checks "Your server responds with HTTP 2xx"; "If the test fails, the webhook remains inactive". Newly created webhooks are "inactive by default".
- **Not documented:** the format of the test (dummy) notification.

## 5. Storage / retention

- **Not documented:** a "550 days" figure does not appear in ACI's docs. 550 days (~18 months) may be an internal figure; origin unverified.
- **Documented [R]:** Payment Data "14 months"; "Card retention will be determined by the expiry date. The transaction will remain in the system for 14 months after card expiry."; "Any follow-up transaction extends the retention of the original payment with another 14 months."
- **Documented [R]:** files (bank, clearing, batch, reconciliation, settlement, token import) "3 months" (transaction export files 28 days; backup and monitoring files 1 month); "Audit logged events" "14 months".
- **Documented [W]:** failed webhooks "stored for 30 days, then purged".
- **Documented [W]:** Fields setting: "ALL (full payload) or NON_CUSTOMER_DATA (exclude sensitive fields)". The payload examples include card `bin`, `last4Digits`, `holder`, and customer name and `email`; no full card number appears.
- **Documented [F]:** PCI DSS is mentioned only for TLS: "For PCI DSS compliance, SSL (Secure Socket Layers) connections using untrusted certificates are not allowed."
- **Inferred (not ACI):** an encrypted body is still personal data under GDPR (storage limitation applies). NON_CUSTOMER_DATA reduces risk. Replay needs the old webhook secret kept as long as the bodies, so store a key version with each stored object.
- **Not documented:** ACI says nothing on compliance for keeping encrypted bodies. Needs the owner's compliance/DPO view.

## Open questions

1. Notification / delivery id: none documented.
2. Is a retry byte-identical (same IV/tag) or re-encrypted?
3. Exact retry attempt count.
4. Does the endpoint ever auto-disable (threshold)?
5. Format of the "Click to Test" dummy notification.
6. Origin of the 550-day retention figure.
