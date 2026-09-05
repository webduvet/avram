# Connect webhook events

Supported notification event types, decrypted envelope shape, payload-shape notes, and concrete examples.

**Sources**

| Local file | Source URL | updatedAt |
| --- | --- | --- |
| Connect webhooks guide | https://docs.bankingcircleconnect.com/docs/webhooks | `2026-05-22T08:13:50.000Z` |
| Connect payload-examples guide | https://docs.bankingcircleconnect.com/docs/payload-examples | `2026-07-14T11:24:02.000Z` |
| subscriptionEvent OpenAPI | https://docs.bankingcircleconnect.com/reference/post_api-v1-notificationselfservice-subscriptionevent | `2026-06-08T12:02:48.000Z` |

## Supported event types

### Payment event types

| Event | Enum | Description | Supported targets | Body property |
| --- | --- | --- | --- | --- |
| IncomingPaymentProcessed | 1 | Account successfully funded (includes internal credit transfers) | Account (0), Company (1) | `payment` |
| OutgoingPaymentRejected | 2 | Outgoing payment rejected; status `Rejected` | Account (0), Company (1) | `payment` |
| OutgoingPaymentProcessed | 3 | Outgoing payment processed (includes internal debit); status `Processed` | Account (0), Company (1) | `payment` |
| MissingFunding | 4 | Insufficient balance; auto-executes when funded or rejected after 2 business days; status `MissingFunding` | Account (0), Company (1) | `payment` |
| Reversed | 5 | Outgoing payment reversed; status `Reversed` | Account (0), Company (1) | `payment` |
| OutgoingPaymentBooked | 6 | Outgoing payment booked on the account | Account (0), Company (1) | `payment` |
| PaymentRouting | 7 | Outgoing payment routed via another scheme; details in `routingStatus` | Account (0), Company (1) | `payment` |
| IncomingPaymentBooked | 8 | Incoming payment booked on the account | Account (0), Company (1) | `payment` |
| OutgoingDirectDebitPendingProcessing | 10 | Third-party initiated outgoing (e.g. Bacs / SEPA DD) pending processing | Account (0), Company (1) | `payment` |
| PaymentStatus | 13 | Status or sub-status update on a payment instruction | Account (0), Company (1) | **`payload`** |

### Other event types

| Event | Enum | Description | Supported targets | Body property |
| --- | --- | --- | --- | --- |
| AccountHolderVerification | 9 | Response to an Account Verification Request | CompanyGroup (2) | (see payload examples) |
| CaseEvents | 11 | Case created or updated | Company (1) | see spelling note below |
| AgencyBankingWhitelistResult | 12 | Result of Agency Banking whitelist request | Company (1) | **`payload`** |
| MandateStatus | 14 | Present in OpenAPI enum for subscriptionEvent (not listed in the webhooks guide table) | — | — |

Target type enums: `0` = Account, `1` = Company, `2` = CompanyGroup.

## Envelope after decrypt

Wire format is encrypted `application/octet-stream`. After AES-256-GCM decrypt, the body is a JSON object with a `notifications` array. Headers on the HTTP request:

| Header | Description |
| --- | --- |
| SubscriptionVersion | Subscription / payload version |
| Nonce | AES-GCM nonce |
| AuthenticationTag | AES-GCM auth tag |
| Checksum | Base64 SHA-256 of decrypted UTF-8 body |

Envelope fields per notification:

| Field | Description |
| --- | --- |
| eventId | UUID of this notification |
| subscriptionId | Parent subscription UUID |
| subscriptionEventId | Subscription-event configuration UUID |
| notificationType | Event name string |
| timestamp | When the notification was triggered |
| payment **or** payload | Event-specific object (name depends on event type) |

Minimal envelope shape (IDs redacted):

```json
{
  "notifications": [
    {
      "subscriptionId": "{subscription-id}",
      "subscriptionEventId": "{subscription-event-id}",
      "eventId": "{event-id}",
      "notificationType": "IncomingPaymentProcessed",
      "timestamp": "2023-09-23T14:53:52.8581798Z",
      "payment": { }
    }
  ]
}
```

Parsers should be flexible: property order may change and new properties may appear.

## Notable payload-shape notes

- **`payment` vs `payload`:** Most payment events nest details under `payment`. **`PaymentStatus`** and **`AgencyBankingWhitelistResult`** use **`payload`** instead.
- **CaseEvents spelling:** Subscription / enum name is `CaseEvents`, but the payload-examples sample uses `"notificationType": "CasesEvents"` (extra `s`) and nests case data under `payment`. Treat both spellings carefully in parsers.
- **Returned payments:** No dedicated return status or webhook. Returns appear as **IncomingPaymentProcessed** with `"return": true`. Remittance often contains `RETURN OF PAYMENT` plus original reference and reason (e.g. account closed).
- **OutgoingPaymentBooked + reversal:** In a reversal scenario, two `OutgoingPaymentBooked` webhooks fire (initial book + later reverse). Account and transaction references match; amount reflects balance effect.
- **Batching:** Multiple notifications can arrive in one message (`maxNotificationsPerMessage` 5–1000).

## Example 1 — IncomingPaymentProcessed (return)

Shortened from `payload-examples.md`. Field names preserved; UUIDs redacted.

```json
{
  "notifications": [
    {
      "subscriptionId": "{subscription-id}",
      "subscriptionEventId": "{subscription-event-id}",
      "eventId": "{event-id}",
      "notificationType": "IncomingPaymentProcessed",
      "timestamp": "2023-09-23T14:53:52.8581798Z",
      "payment": {
        "paymentId": "{payment-id}",
        "transactionReference": "010F103220030044",
        "classification": "Incoming",
        "subClassification": "Internal",
        "isTraceable": false,
        "status": "Processed",
        "processedTimestamp": "2023-09-23T15:33:51.0133333+00:00",
        "latestStatusChangedTimestamp": "2023-09-23T15:33:51.0133333+00:00",
        "booked": null,
        "reversalBooked": null,
        "return": true,
        "paymentRail": "Internal",
        "errors": null,
        "transfer": {
          "debtorAccount": {
            "accountNumber": "0000026804",
            "accountIban": "DE42700700100000012345",
            "account": "DE42700700100000012345",
            "financialInstitution": "DEUTDEMMXXX",
            "country": null
          },
          "debtorName": "Funding of test account",
          "amount": { "currency": "GBP", "amount": 145.25 },
          "valueDate": "2021-09-23T00:00:00+00:00",
          "chargeBearer": "Ben",
          "remittanceInformation": {
            "line1": "RETURN OF PAYMENT",
            "line2": "010F311252940IZD",
            "line3": null,
            "line4": null
          },
          "creditorAccount": {
            "accountNumber": null,
            "accountIban": "DK1089000049910051",
            "account": "DK1089000049910051",
            "financialInstitution": "SXPYDKKKXXX",
            "country": "DK"
          },
          "creditorProxyAccount": {
            "service": "PayId",
            "type": "EMAL",
            "value": "test@example.com"
          }
        },
        "creditorInformation": {
          "accountId": "{account-id}"
        }
      }
    }
  ]
}
```

## Example 2 — PaymentStatus (`payload`)

```json
{
  "notifications": [
    {
      "subscriptionId": "{subscription-id}",
      "subscriptionEventId": "{subscription-event-id}",
      "eventId": "{event-id}",
      "notificationType": "PaymentStatus",
      "timestamp": "2023-05-05T20:34:33.0350148Z",
      "payload": {
        "EndToEndId": "Tc188435CPS260528110949807",
        "InstructionUetr": "{uetr}",
        "AccountServicerReference": "010F209261480006",
        "AcceptanceDateTime": "2026-05-28T11:09:50.1947352Z",
        "TransactionStatus": "PendingProcessing",
        "StatusReasonInformation": [
          {
            "Reason": "AB08",
            "AdditionalInformation": "Offline Creditor Agent. Scheme: FasterPayments. Attempts remaining: 2."
          }
        ],
        "TransactionReference": {
          "InstructedAmount": { "Currency": "EUR", "Amount": 7.92 },
          "DebtorAccount": {
            "Iban": "DK1889000000010682",
            "AccountNumber": "0000010682",
            "AccountUuid": "{account-id}"
          },
          "DebtorAgent": { "Bic": "SXPYDKKKXXX", "Ncc": null, "Country": "DK" },
          "CreditorAccount": { "Iban": "DE95440400370347777500", "AccountNumber": null },
          "CreditorAgent": { "Bic": "COBADEFFXXX", "Ncc": null, "Country": "DE" }
        }
      }
    }
  ]
}
```

## Example note — CaseEvents / CasesEvents

```json
{
  "notifications": [
    {
      "subscriptionId": "{subscription-id}",
      "subscriptionEventId": "{subscription-event-id}",
      "eventId": "{event-id}",
      "notificationType": "CasesEvents",
      "timestamp": "2025-12-08T10:50:49.9866667Z",
      "payment": {
        "eventType": "CASE_CREATED",
        "caseId": "{case-id}",
        "caseType": "RFI",
        "status": "OPEN",
        "deadline": "2025-12-03T14:23:05.5826095Z",
        "links": {
          "self": "/api/v1/cases/rfi/{case-id}"
        }
      }
    }
  ]
}
```
