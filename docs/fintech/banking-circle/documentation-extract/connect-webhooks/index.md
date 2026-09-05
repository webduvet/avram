# Connect webhooks

Overview of Banking Circle Connect webhook delivery: encrypted push notifications for payment and related events. Webhooks are not a substitute for reconciliation; use the reconciliation report for that purpose.

## Subsections

- [Events](./events.md) — supported event types, envelope, payload notes and examples
- [Subscriptions](./subscriptions.md) — notification self-service API (create/list/get/update/delete/activate/deactivate, subscription events, clienttest) and retry strategy

## Setup summary

Sources: `receiving-webhooks.md` (`updatedAt: 2025-09-17T15:24:42.000Z`), `webhooks.md` (`updatedAt: 2026-05-22T08:13:50.000Z`).

### Endpoint requirements

- HTTPS URL; IPv4
- Accept HTTP POST with encrypted body as `Content-Type: application/octet-stream` (JSON after decrypt)
- Payload size up to 50MB
- Respond with HTTP `2xx` within **10 seconds** (other codes trigger retry)

### Security

- TLS 1.2
- AES-256-GCM encryption (32-character encryption key supplied by the subscriber)
- SHA-256 checksum in request header (verify against decrypted UTF-8 body)
- Headers involved in decrypt/verify: `Nonce`, `AuthenticationTag`, `Checksum`, `SubscriptionVersion`
- Optional mutual TLS (mTLS) by arrangement

### SLA / delivery notes

- 100% send-rate guarantee; receipt depends on client endpoint availability/configuration
- Duplicates possible — implement idempotency
- Failed non-2xx responses enter the retry sequence (see [Subscriptions — retry strategy](./subscriptions.md#retry-strategy))

### IP allowlists

Firewall must allow Banking Circle source IPs.

#### Production

```
51.104.183.88, 51.104.183.94, 51.104.183.101, 51.104.183.103, 51.104.183.106, 51.104.183.115, 51.104.183.229, 20.54.48.95, 20.54.48.105, 20.54.48.113, 20.54.48.122, 20.54.48.130, 20.67.184.104, 20.67.185.55, 20.67.185.96, 20.67.185.153, 20.67.185.206, 20.67.185.221, 20.67.185.228, 20.67.185.250, 20.67.186.42, 20.67.186.48, 20.67.186.50, 20.67.186.64, 20.54.48.159, 20.54.48.196, 20.54.48.210, 20.54.48.219, 20.54.48.241, 20.54.49.10, 20.50.64.12, 20.73.113.63, 20.76.58.170, 20.76.58.202, 20.76.59.51, 20.76.59.68, 20.76.59.96, 20.73.113.113, 20.76.59.149, 20.76.59.189, 20.76.59.236, 20.73.118.112, 20.76.56.90, 20.73.113.107, 20.76.60.207, 20.76.60.223, 20.76.61.55, 20.73.117.222, 20.76.61.195, 20.76.57.112, 20.73.116.227, 20.76.57.135, 20.76.61.202, 51.138.13.120, 51.138.97.100, 51.138.98.69, 51.124.151.146, 51.138.100.153, 51.138.101.10, 51.138.102.72, 20.50.154.139, 20.50.2.55, 20.157.111.19, 135.236.145.9, 135.236.144.255, 20.157.115.2, 135.236.66.216, 135.236.67.38
```

#### Sandbox

```
20.67.201.6, 20.67.201.16, 20.67.201.23, 20.67.201.40, 20.67.201.48, 20.67.201.53, 20.67.201.72, 20.67.201.99, 20.67.201.117, 20.67.202.197, 20.67.202.200, 20.67.202.212, 20.67.202.220, 20.67.202.248, 20.67.203.10, 20.67.203.20, 20.67.203.26, 20.67.203.39, 20.67.203.47, 20.67.203.61, 20.67.203.66, 20.67.200.198, 20.67.200.210, 20.67.200.229, 20.67.200.248, 20.67.200.252, 20.67.201.3, 20.67.201.36, 20.67.201.50, 20.67.201.85, 20.50.64.11, 52.156.253.199, 52.156.253.140, 135.236.114.55, 135.236.114.124
```

## Source URLs

- https://docs.bankingcircleconnect.com/docs/receiving-webhooks
- https://docs.bankingcircleconnect.com/docs/webhooks
