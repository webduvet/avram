# Connect webhook subscriptions

Notification self-service API for webhook subscriptions and subscription events, plus retry strategy.

**Guide sources:** `webhook-subscriptions.md` (`updatedAt: 2026-06-30T07:48:42.000Z`), `webhook-retry-strategy.md` (`updatedAt: 2025-11-18T10:15:06.000Z`).

**Reference OpenAPI (local mirror):** notification self-service OpenAPI pages in the local Connect reference mirror (paths listed per endpoint). Live base URL: `https://www.bankingcircleconnect.com` (sandbox clienttest: `https://sandbox.bankingcircleconnect.com`).

## Shared rules

| Rule | Detail |
| --- | --- |
| Status enum | `0` = None, `1` = Inactive, `2` = Active, `4` = Retired |
| `maxNotificationsPerMessage` | Integer **5–1000** (default commonly 1000) |
| Unique URL | Subscriptions **cannot** share identical endpoint URLs |
| Encryption key | 32-character key provided by the client; responses typically show `*Hidden*` |
| Concurrency | Mutating calls require `If-Match` header = current `rowVersion` from GET |
| Auth | Bearer token |
| Event uniqueness | Same `eventType` only once per subscription; cannot mix Account and Company targets on one event |
| Target IDs | From `GET /api/v1/accounts` — company GUID → `ownedByCompanyId`, account GUID → `accountId`, passed as `targetId` / `targetIds` |

Dashboard UI also exposes Active / Active-with-failures / Inactive cards, restart delivery, and failed-delivery status codes 500 / 404 / 401 / 400.

---

## POST `/api/v1/notificationselfservice/subscription`

Create a subscription.

- **Source:** `post_api-v1-notificationselfservice-subscription.md` — `updatedAt: 2026-06-01T07:44:00.000Z`
- **URL:** https://docs.bankingcircleconnect.com/reference/post_api-v1-notificationselfservice-subscription
- **Required body:** `encryptionKey`, `endpoint`, `status`
- **Optional:** `email`, `maxNotificationsPerMessage` (5–1000)
- **Headers:** `Authorization: Bearer {token}`, `Content-Type: application/json`
- Set `status` to `2` for active on create, or `1` for inactive

### Request example

```json
{
  "endpoint": "https://www.example.com/webhooks/bc",
  "status": 2,
  "encryptionKey": "{32-char-encryption-key}",
  "email": "alerts@example.com",
  "maxNotificationsPerMessage": 1000
}
```

### Response example (`200` — SubscriptionCommandResult)

```json
{
  "id": "{subscription-id}",
  "endpoint": "https://www.example.com/webhooks/bc",
  "isActive": true,
  "mtlsEnabled": false,
  "status": 2,
  "statusMessage": "Subscription successfully created",
  "rowVersion": "AAAAAAAAAAA=",
  "version": 1,
  "encryptionKey": "*Hidden*",
  "email": "alerts@example.com",
  "maxNotificationsPerMessage": 1000,
  "subscriptionEvents": null
}
```

Note: OpenAPI `statusMessage` description mentions deactivation “after **10** retries”, while the dedicated retry guide documents **11** retries (see [Retry strategy](#retry-strategy)).

---

## GET `/api/v1/notificationselfservice/subscription`

List all subscriptions (paged).

- **Source:** `get_api-v1-notificationselfservice-subscription.md` — `updatedAt: 2026-06-01T07:44:00.000Z`
- **Query:** `PageNumber` (min 1, default 1), `PageSize` (schema min 1 / max 100 in OpenAPI, description mentions up to 5000; default 50)
- **Headers:** Bearer

### Response shape (per item)

Same fields as create result, plus nested `subscriptionEvents[]` with `id`, `eventType`, `isActive`, `subscriptionId`, `status`, `rowVersion`, `subscriptionEventTargetDetails[]` (`id`, `targetId`, `targetType`).

```json
{
  "id": "{subscription-id}",
  "endpoint": "https://www.example.com/webhooks/bc",
  "isActive": true,
  "mtlsEnabled": false,
  "status": 2,
  "statusMessage": "Subscription successfully created",
  "rowVersion": "AAAAAAAAAAA=",
  "version": 1,
  "email": "alerts@example.com",
  "maxNotificationsPerMessage": 1000,
  "subscriptionEvents": [
    {
      "id": "{subscription-event-id}",
      "eventType": "IncomingPaymentProcessed",
      "isActive": true,
      "subscriptionId": "{subscription-id}",
      "status": 2,
      "rowVersion": "AAAAAAAAAAA=",
      "subscriptionEventTargetDetails": [
        {
          "id": "{target-detail-id}",
          "targetId": "{company-or-account-id}",
          "targetType": 1
        }
      ]
    }
  ]
}
```

---

## GET `/api/v1/notificationselfservice/subscription/{id}`

Get one subscription and its events.

- **Source:** `get_api-v1-notificationselfservice-subscription-id.md` — `updatedAt: 2026-06-01T07:44:00.000Z`
- **Path:** `id` (UUID, required)
- **Responses:** `200` SubscriptionQueryResult; `401`; `404` NotificationSelfServiceErrorResponse; `500`

### Response example

```json
{
  "id": "{subscription-id}",
  "endpoint": "https://www.example.com/webhooks/bc",
  "isActive": true,
  "mtlsEnabled": false,
  "status": 2,
  "statusMessage": "Subscription successfully created",
  "rowVersion": "AAAAAAAAAAA=",
  "version": 1,
  "email": "alerts@example.com",
  "maxNotificationsPerMessage": 1000,
  "subscriptionEvents": [
    {
      "id": "{subscription-event-id}",
      "eventType": "CaseEvents",
      "isActive": true,
      "subscriptionId": "{subscription-id}",
      "status": 2,
      "rowVersion": "AAAAAAAAAAA=",
      "subscriptionEventTargetDetails": [
        {
          "id": "{target-detail-id}",
          "targetId": "{company-id}",
          "targetType": 1
        }
      ]
    }
  ]
}
```

---

## PUT `/api/v1/notificationselfservice/subscription/{id}`

Update encryption key, endpoint URL, email, max notifications, and/or mTLS flag.

- **Source:** `put_api-v1-notificationselfservice-subscription-id.md` — `updatedAt: 2026-06-01T07:44:00.000Z`
- **Headers:** `If-Match` (required) = `rowVersion`; Bearer
- **Body (UpdateSubscriptionDto):** all optional — `encryptionKey`, `endpoint`, `email`, `maxNotificationsPerMessage` (5–1000), `mtlsEnabled`

### Request example

```http
PUT /api/v1/notificationselfservice/subscription/{subscription-id}
If-Match: AAAAAAAAAAA=
Content-Type: application/json
```

```json
{
  "endpoint": "https://www.example.com/webhooks/bc-v2",
  "email": "alerts@example.com",
  "maxNotificationsPerMessage": 100,
  "encryptionKey": "{32-char-encryption-key}",
  "mtlsEnabled": false
}
```

### Response example (`200`)

```json
{
  "id": "{subscription-id}",
  "endpoint": "https://www.example.com/webhooks/bc-v2",
  "isActive": true,
  "mtlsEnabled": false,
  "status": 2,
  "statusMessage": "Subscription successfully created",
  "rowVersion": "AAAAAAAAAAE=",
  "version": 1,
  "encryptionKey": "*Hidden*",
  "email": "alerts@example.com",
  "maxNotificationsPerMessage": 100,
  "subscriptionEvents": null
}
```

`rowVersion` changes on each retry attempt as well as on updates — a concurrent retry can invalidate an in-flight PUT.

---

## DELETE `/api/v1/notificationselfservice/subscription/{id}`

Delete a subscription (irreversible).

- **Source:** `delete_api-v1-notificationselfservice-subscription-id.md` — `updatedAt: 2026-06-01T07:44:00.000Z`
- **Headers:** `If-Match` (required); Bearer
- **Response:** `200` OK (empty); `401`; `404`; `500`

```http
DELETE /api/v1/notificationselfservice/subscription/{subscription-id}
If-Match: AAAAAAAAAAA=
Authorization: Bearer {token}
```

---

## PUT `/api/v1/notificationselfservice/subscription/{id}/activate`

Activate an inactive subscription.

- **Source:** `put_api-v1-notificationselfservice-subscription-id-activate.md` — `updatedAt: 2026-06-01T07:44:00.000Z`
- **Headers:** `If-Match` (required); Bearer
- **Body:** none
- **Response:** `200` OK

```http
PUT /api/v1/notificationselfservice/subscription/{subscription-id}/activate
If-Match: AAAAAAAAAAA=
```

Pending notifications queued before deactivation are delivered after reactivation; events that occurred while inactive are not backfilled.

---

## PUT `/api/v1/notificationselfservice/subscription/{id}/deactivate`

Deactivate a subscription.

- **Source:** `put_api-v1-notificationselfservice-subscription-id-deactivate.md` — `updatedAt: 2026-06-01T07:44:00.000Z`
- **Headers:** `If-Match` (required); Bearer
- **Response:** `200` OK

```http
PUT /api/v1/notificationselfservice/subscription/{subscription-id}/deactivate
If-Match: AAAAAAAAAAA=
```

---

## POST `/api/v1/notificationselfservice/subscriptionEvent`

Add an event to a subscription (create subscription first).

- **Source:** `post_api-v1-notificationselfservice-subscriptionevent.md` — `updatedAt: 2026-06-08T12:02:48.000Z`
- **Required body:** `eventType`, `subscriptionId`, `targetType`
- **Optional:** `targetIds` (array of UUIDs)
- **Headers:** Bearer; JSON body

Guide note in this reference: after the **tenth** unsuccessful retry the subscription is deactivated — conflicts with the 11-row retry table (see below).

### Request example

```json
{
  "subscriptionId": "{subscription-id}",
  "eventType": "IncomingPaymentProcessed",
  "targetType": 1,
  "targetIds": ["{company-id}"]
}
```

### Response example (`200` — SubscriptionEventCommandResult)

```json
{
  "id": "{subscription-event-id}",
  "eventType": "IncomingPaymentProcessed",
  "isActive": true,
  "subscriptionId": "{subscription-id}",
  "status": 2,
  "rowVersion": "AAAAAAAAAAA=",
  "subscriptionEventTargetDetails": [
    {
      "id": "{target-detail-id}",
      "targetId": "{company-id}",
      "targetType": 1
    }
  ]
}
```

Older guide text stated targets could not be edited in place (delete + recreate). The replace-targets endpoint below supersedes that for target lists.

---

## DELETE `/api/v1/notificationselfservice/subscriptionEvent/{id}`

Delete a subscription event.

- **Source:** `delete_api-v1-notificationselfservice-subscriptionevent-id.md` — `updatedAt: 2026-06-01T07:44:00.000Z`
- **Path:** subscription-event `id`
- **Headers:** `If-Match` (required) = event `rowVersion`; Bearer
- **Response:** `200` OK

```http
DELETE /api/v1/notificationselfservice/subscriptionEvent/{subscription-event-id}
If-Match: AAAAAAAAAAA=
```

---

## PUT `/api/v1/notificationselfservice/subscriptionEvent/{id}/targets`

Replace all targets on a subscription event.

- **Source:** `put_api-v1-notificationselfservice-subscriptionevent-id-targets.md` — `updatedAt: 2026-08-28T07:26:05.000Z`
- **Headers:** `If-Match` (required); Bearer
- **Required body:** `targetIds` (minItems 1), `targetType` (0/1/2)

### Request example

```json
{
  "targetType": 0,
  "targetIds": ["{account-id-1}", "{account-id-2}"]
}
```

### Response example (`200` — SubscriptionEventCommandResultV2)

```json
{
  "id": "{subscription-event-id}",
  "eventType": "IncomingPaymentProcessed",
  "subscriptionId": "{subscription-id}",
  "status": "Active",
  "rowVersion": "AAAAAAAAAAE=",
  "subscriptionEventTargetDetails": [
    {
      "id": "{target-detail-id}",
      "targetId": "{account-id-1}",
      "targetType": 0
    }
  ]
}
```

(V2 response uses string status `Active` / `Inactive` rather than numeric enum.)

---

## POST `/api/v1/notificationselfservice/clienttest/{id}` (sandbox)

Trigger mocked notifications for a subscription to test the endpoint.

- **Source:** `post_api-v1-notificationselfservice-clienttest-id.md` — `updatedAt: 2025-09-17T15:21:47.000Z`
- **Server:** `https://sandbox.bankingcircleconnect.com` only
- **Path:** subscription `id`
- **Body:** none
- **Response:** `200` with information about triggered subscription events; `401`; `404`; `500`

```http
POST https://sandbox.bankingcircleconnect.com/api/v1/notificationselfservice/clienttest/{subscription-id}
Authorization: Bearer {token}
```

---

## Retry strategy

Source: `webhook-retry-strategy.md` — `updatedAt: 2025-11-18T10:15:06.000Z`  
URL: https://docs.bankingcircleconnect.com/docs/webhook-retry-strategy

Successful delivery = HTTP `2xx`. Once acknowledged with 2xx, a notification is not resent. Non-2xx starts exponential backoff. After the final unsuccessful attempt the subscription is deactivated, a deactivation email is sent (if configured), pending notifications are retained for delivery after reactivation, and events during deactivation are not notified.

### Retry schedule (11 rows)

| Retry | Time after last retry | Email notification |
| --- | --- | --- |
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
| 11th | 48:00:00 | Deactivation email |

### 10 vs 11 conflict

- Retry guide: **up to 11 retry attempts** then deactivate.
- OpenAPI `statusMessage` text and `subscriptionEvent` guide prose: deactivation after **10** unsuccessful retries.

Treat **11** as the schedule of record from `webhook-retry-strategy.md`, and note the conflicting “10” wording elsewhere until clarified with Banking Circle.
