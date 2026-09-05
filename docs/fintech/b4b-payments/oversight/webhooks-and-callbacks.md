# Oversight webhooks and callbacks

Sources: `beneficiaries-and-payments.md` (webhook sections), `beneficiaries-and-payments-KEY-EXTRACTS.md`, `webhook-signature-verification.md`, `webhooks-overview.md`.

## Oversight payment callbacks (`callback_url`)

Provided on `POST /payments`. Oversight POSTs on regulatory-phase status changes. Final statuses from Oversight: **`B4BTMApproved`** (handed to BC) or **`B4BFailed`** (will not send).

### Stops at handoff

Settlement after handoff is **not** covered by Oversight callbacks. Subscribe to Banking Circle payment notifications and correlate on `banking_circle_api_response.paymentId` from the final Oversight webhook.

### Payload

Full payment object as held by Oversight, including BC-shaped `payload`, and when present `banking_circle_api_response`:

```json
{
  "id": "{payment_id}",
  "status": "B4BTMApproved",
  "payload": { },
  "banking_circle_api_response": {
    "paymentId": "{banking_circle_payment_id}",
    "status": "PendingProcessing"
  }
}
```

| Field | When present |
|---|---|
| `id` | Always — Oversight payment UUID |
| `status` | Always — one of the six Oversight statuses |
| `payload` | Always — BC-shaped body as submitted |
| `banking_circle_api_response` | After handoff attempt; includes BC `paymentId`, or BC error on handoff refusal |

Correlation: own systems via `external_ref` (echoed on related lookups); BC via `banking_circle_api_response.paymentId`.

### Delivery semantics (payments)

- **Transport:** HTTPS POST, `Content-Type: application/json`
- **Signing:** **Not currently signed** for Oversight outbound payment webhooks. Use TLS; validate identity of the payment (e.g. known `id` / `external_ref`); IP allow-listing recommended
- **Retries:** Polynomial backoff on 5xx or network error; stop after roughly a day
- **Replay:** No manual replay endpoint; fall back to `GET /payments/{id}`
- **Idempotency:** Same status may arrive more than once — treat as current state
- **Ordering:** Not guaranteed
- **Ack:** Any `2xx` (`200`/`201`/`202`/`204`)

## Oversight beneficiary callbacks

`callback_url` on `POST /beneficiaries`. Fired when consolidated sanctions status changes.

```json
{
  "external_ref": "{YOUR_BENEFICIARY_REF}",
  "sanctions_status": "pass"
}
```

`sanctions_status`: `pass` | `review` | `fail`. Only `pass` permits payments. Status can move later under continuous screening.

Delivery semantics match payment callbacks (unsigned Oversight outbound; TLS; polynomial backoff ~1 day; no replay; idempotent re-delivery; ordering not guaranteed; any `2xx`). Missed events → `GET /beneficiaries/{id}`.

Person create also accepts a `callback_url` for person-level sanctions updates; person outcomes do **not** gate payments (informational).

## Extra post-handoff callback — undocumented

Any additional Oversight callback **after** handoff (beyond documented `B4BTMApproved` / `B4BFailed` terminal webhook behaviour) is **not** product documentation in these sources. Do not invent behaviour. Track open questions under [`../questions-and-undocumented-features.md`](../questions-and-undocumented-features.md). For settlement events, use Banking Circle webhooks/reports.

## On-platform webhook signing (not Oversight payment callbacks)

`webhook-signature-verification.md` and `webhooks-overview.md` describe the **platform** webhook offering (e.g. card transaction push notifications), which **does** use signature headers. That mechanism must **not** be assumed for Oversight payment/beneficiary `callback_url` posts (explicitly unsigned in the Oversight guide).

### Platform signature headers (for contrast / other products)

| Header | Meaning |
|---|---|
| `X-B4b-Webhook-Signature` | Base64-encoded signature |
| `X-B4b-Webhook-Signature-Timestamp` | UNIX timestamp of signature generation |
| `X-B4b-Webhook-Signature-Version` | Currently `"1"` |

Public key: `https://b4bpayments.com/.well-known/public-keys.json` at `.public_keys.webhook_signing` (PEM). ECDSA key pair; verify `SHA256` over `{timestamp}.{json_body}`.

Sample verification (Ruby, from source; unchanged procedure):

```ruby
require "base64"
require "net/http"
require "openssl"

def verify_webhook_signature(request_body:, timestamp_header:, signature_header:)
  public_keys_uri = URI("https://b4bpayments.com/.well-known/public-keys.json")
  public_keys_response = JSON.parse(Net::HTTP.get(public_keys_uri))
  public_key_pem = public_keys_response.dig("public_keys", "webhook_signing")
  public_key = OpenSSL::PKey.read(public_key_pem)

  signature = Base64.strict_decode64(signature_header)
  timestamped_message = "#{timestamp_header}.#{request_body}"
  public_key.verify("SHA256", signature, timestamped_message)
end
```

### Platform webhook overview notes

- HTTPS URL with valid SSL; POST JSON; acknowledge with HTTP 200
- Source IPs can be provided for allow-listing
- Partial fault tolerance: reconnect with increasing intervals; long outages may miss events
- See also readme reference `post_webhooks-card-transaction` for card webhook bodies

## After handoff (BC side)

1. Ensure a BC webhook subscription for payment status.
2. Match BC `paymentId` to Oversight’s `banking_circle_api_response.paymentId`.
3. Use BC payload for settlement state and `paymentRail`.

If real-time settlement is not required, BC reconciliation reports include settled payments by date (see [reports.md](reports.md)).
