# Setup webhooks

Practical setup for Banking Circle Connect webhooks. Source of truth is the Connect docs (password-gated). Help Centre articles are secondary and sometimes stale.

Official: [Webhooks](https://docs.bankingcircleconnect.com/docs/webhooks) · [Setting up Webhooks](https://docs.bankingcircleconnect.com/docs/receiving-webhooks) · [Webhook subscriptions](https://docs.bankingcircleconnect.com/docs/webhook-subscriptions) · [Webhook retry strategy](https://docs.bankingcircleconnect.com/docs/webhook-retry-strategy)

## Two surfaces, same object

Subscribe via **Notification Self Service API** or **Client Portal → Webhook subscriptions**. They manage the same subscriptions.

| | Sandbox | Production |
|---|---|---|
| API | `https://sandbox.bankingcircleconnect.com` | `https://www.bankingcircleconnect.com` (note `www`) |
| Auth | `https://authorizationsandbox.bankingcircleconnect.com/` | `https://authorization.bankingcircleconnect.com/` |
| Credentials | separate certs / users / subscriptions | separate |
| Source IPs | sandbox list below | production list below |

A sandbox subscription never receives production events, and vice versa.

Webhook CRUD uses the same M2M user + client cert + Bearer JWT as the rest of Connect. Inbound POSTs do **not** use that Bearer token.

## Create subscription ≠ add events

Creating a subscription does not subscribe any events. Events are a second call.

| Action | Method | Path |
|---|---|---|
| Create subscription | POST | `/api/v1/notificationselfservice/subscription` |
| List | GET | `/api/v1/notificationselfservice/subscription` |
| Get one | GET | `/api/v1/notificationselfservice/subscription/{id}` |
| Update key / URL / email / batch / mTLS | PUT | `/api/v1/notificationselfservice/subscription/{id}` + `If-Match` |
| Delete | DELETE | `/api/v1/notificationselfservice/subscription/{id}` + `If-Match` |
| Activate | PUT | `/api/v1/notificationselfservice/subscription/{id}/activate` + `If-Match` |
| Deactivate | PUT | `/api/v1/notificationselfservice/subscription/{id}/deactivate` + `If-Match` |
| Add event | POST | `/api/v1/notificationselfservice/subscriptionEvent` |
| Delete event | DELETE | `/api/v1/notificationselfservice/subscriptionEvent/{id}` + `If-Match` |
| Replace event targets | PUT | `/api/v1/notificationselfservice/subscriptionEvent/{id}/targets` + `If-Match` |
| Sandbox test | POST | `/api/v1/notificationselfservice/clienttest/{id}` (**sandbox host only**) |

Create required: `encryptionKey`, `endpoint`, `status`. Optional: `email`, `maxNotificationsPerMessage` (`5`–`1000`, default `1000`).

Status: `1` Inactive, `2` Active (`0` None, `4` Retired). Set `2` if it should send immediately.

Add-event required: `eventType`, `subscriptionId`, `targetType`. Optional `targetIds` (account / company / company-group GUIDs). Cannot mix companies and bank accounts on one event. Cannot subscribe the same event twice on one subscription.

No limit on number of subscriptions; **URLs must be unique**.

All mutating calls need a current `If-Match: {rowVersion}`. `rowVersion` **changes on every delivery retry**, so GET immediately before PUT.

## Endpoint requirements

From [Setting up Webhooks](https://docs.bankingcircleconnect.com/docs/receiving-webhooks):

- **HTTPS** + **TLS 1.2**. HTTP is not documented as allowed. Valid certificate. Public internet.
- Accept **HTTP POST** and **binary** `Content-Type: application/octet-stream` (the live body is ciphertext, not JSON).
- IPv4.
- Payload up to **50MB**.
- Respond with **`2xx` within 10 seconds**. Safer to return **`200`**: create-subscription prose emails on non-`200`, while the retry page says any `2xx`. Response body unspecified.
- Non-`2xx` (and implied timeouts) are failed delivery and start retries. See [Troubleshooting](troubleshooting.md) for the table.

Ack **before** decrypting. Returning `401` on a decrypt failure is a failed delivery.

## Encryption (not an HMAC)

This is **AES-256-GCM payload encryption**, not a detached signature of the raw body.

- **32-character** key, chosen at create, rotatable on update. GET responses hide it (`*Hidden*`). Store it yourself.
- Integrity: SHA-256 of decrypted plaintext, Base64, compared to the checksum header.
- Documented logical headers (wire spelling **unspecified**; unofficial SDKs use `X-Bc-*` — unverified):

| Header | Role | Length check in troubleshooting |
|---|---|---|
| `Checksum` | SHA-256 of decrypted body, Base64 | 44 chars |
| `Nonce` | AES-GCM nonce, Base64 | 16 chars |
| `AuthenticationTag` | AES-GCM tag, Base64 | 24 chars |
| `SubscriptionVersion` | subscription / payload version | example `1` |

Official Java/.NET samples decrypt as **UTF-16LE** then checksum **UTF-8** bytes of that text. One .NET sample Base64-decodes the key; Java uses raw 32 UTF-8 bytes. Treat key encoding as an unknown until proven against a live POST.

Optional mTLS (`mtlsEnabled` on **update** only) needs certificate exchange with Banking Circle. Default is TLS only.

## Source IP allowlists

Inbound to *the webhook URL*. Sandbox and production are **different lists**. Azure ranges; re-read the live setup page before tightening a firewall.

Do **not** confuse with Help Centre KA-01131 (your **outbound** IPs to call their API).

**Production**

```
51.104.183.88, 51.104.183.94, 51.104.183.101, 51.104.183.103, 51.104.183.106, 51.104.183.115, 51.104.183.229, 20.54.48.95, 20.54.48.105, 20.54.48.113, 20.54.48.122, 20.54.48.130, 20.67.184.104, 20.67.185.55, 20.67.185.96, 20.67.185.153, 20.67.185.206, 20.67.185.221, 20.67.185.228, 20.67.185.250, 20.67.186.42, 20.67.186.48, 20.67.186.50, 20.67.186.64, 20.54.48.159, 20.54.48.196, 20.54.48.210, 20.54.48.219, 20.54.48.241, 20.54.49.10, 20.50.64.12, 20.73.113.63, 20.76.58.170, 20.76.58.202, 20.76.59.51, 20.76.59.68, 20.76.59.96, 20.73.113.113, 20.76.59.149, 20.76.59.189, 20.76.59.236, 20.73.118.112, 20.76.56.90, 20.73.113.107, 20.76.60.207, 20.76.60.223, 20.76.61.55, 20.73.117.222, 20.76.61.195, 20.76.57.112, 20.73.116.227, 20.76.57.135, 20.76.61.202, 51.138.13.120, 51.138.97.100, 51.138.98.69, 51.124.151.146, 51.138.100.153, 51.138.101.10, 51.138.102.72, 20.50.154.139, 20.50.2.55, 20.157.111.19, 135.236.145.9, 135.236.144.255, 20.157.115.2, 135.236.66.216, 135.236.67.38
```

**Sandbox**

```
20.67.201.6, 20.67.201.16, 20.67.201.23, 20.67.201.40, 20.67.201.48, 20.67.201.53, 20.67.201.72, 20.67.201.99, 20.67.201.117, 20.67.202.197, 20.67.202.200, 20.67.202.212, 20.67.202.220, 20.67.202.248, 20.67.203.10, 20.67.203.20, 20.67.203.26, 20.67.203.39, 20.67.203.47, 20.67.203.61, 20.67.203.66, 20.67.200.198, 20.67.200.210, 20.67.200.229, 20.67.200.248, 20.67.200.252, 20.67.201.3, 20.67.201.36, 20.67.201.50, 20.67.201.85, 20.50.64.11, 52.156.253.199, 52.156.253.140, 135.236.114.55, 135.236.114.124
```

## Testing: there is no portal ping

Portal **Restart delivery** retries **failed** notifications on an **Active** subscription and resets the failed-attempt counter. It is not a synthetic test event.

Sandbox only:

```
POST https://sandbox.bankingcircleconnect.com/api/v1/notificationselfservice/clienttest/{subscriptionId}
```

Docs: a mocked payment matching the subscription is sent, triggering a notification. **200 from this API means “events triggered”, not “the endpoint returned 2xx”.** It does not create an underlying payment. Production has no equivalent. A green sandbox test does not prove production (different IPs, certs, subscription).

## Event types and payload shape

After decrypt, envelope is `notifications[]` with `subscriptionId`, `subscriptionEventId`, `eventId`, `notificationType`, `timestamp`, and either **`payment` or `payload`**. Build a flexible parser; properties can move and new ones can appear.

Idempotency: **`eventId`**. Duplicates may occur. `Processed` and `Booked` are independent and **may arrive in any order**.

### Payment events (target Account `0` or Company `1`)

| Event | Enum | Inner object |
|---|---|---|
| `IncomingPaymentProcessed` | 1 | `payment` |
| `OutgoingPaymentRejected` | 2 | `payment` |
| `OutgoingPaymentProcessed` | 3 | `payment` |
| `MissingFunding` | 4 | `payment` (auto-executes when funded, or rejected after **2 business days**) |
| `Reversed` | 5 | `payment` |
| `OutgoingPaymentBooked` | 6 | `payment` |
| `PaymentRouting` | 7 | `payment` |
| `IncomingPaymentBooked` | 8 | `payment` |
| `OutgoingDirectDebitPendingProcessing` | 10 | `payment` |
| `PaymentStatus` | 13 | **`payload`** (PascalCase ISO-like fields) |

### Other

| Event | Enum | Targets |
|---|---|---|
| `AccountHolderVerification` | 9 | **CompanyGroup (`2`) only** |
| `CaseEvents` | 11 | Company (`1`) |
| `AgencyBankingWhitelistResult` | 12 | Company (`1`) |

OpenAPI also has `MandateStatus` = 14; **not** on the Webhooks guide table. Unspecified whether it is live.

Notes from official examples:

- `PaymentStatus` is easy to miss if the parser only reads `payment`.
- `CaseEvents` examples use `notificationType: "CasesEvents"` (extra **s**).
- Returned payments have no dedicated event; they show up as Incoming Processed, often with `RETURN OF PAYMENT` in remittance info.
- Reversal fires **two** `OutgoingPaymentBooked` notifications.

Help Centre [KA-01118](https://support.bankingcircle.com/article/KA-01118/en-us) is a shorter older set (wrong names, MissingFunding “12 hours”). **Use the Connect docs table.**

## Batching and size

`maxNotificationsPerMessage` 5–1000 (default 1000). Payload up to 50MB.

Webhooks are not a substitute for the [reconciliation report](https://docs.bankingcircleconnect.com/docs/reconciliation-report).
