# Oversight authentication

Sources: `authenticating-with-the-oversight-api.md`, `setting-up-authentication.md` (on-platform contrast only).

## Oversight API — JWT (RS512)

Oversight uses JSON Web Tokens signed with **RS512**. The integrator generates an RSA key pair, shares the **public** key with the provider’s technical team, and receives a **Key ID** (`kid`) to put in the JWT header. The private key signs tokens and must never be shared.

### Key pair generation (documented commands)

```bash
ssh-keygen -t rsa -b 4096 -m PEM -f jwt.key
openssl rsa -in jwt.key -pubout -outform PEM -out jwt.key.pub
```

- `jwt.key` — private; used to sign JWTs on the host.
- `jwt.key.pub` — public; provided to the technical team; returns a Key ID for the `kid` claim.

Passphrase: leave blank if the key must be usable by automated processes without interactive unlock.

### JWT header

| Key | Example | Notes |
|---|---|---|
| `alg` | `RS512` | Required algorithm |
| `typ` | `JWT` | |
| `kid` | `{YOUR_KEY_ID}` | Unique string identifying the registered public key |

### JWT payload

| Key | Example | Notes |
|---|---|---|
| `iat` | `1620345605` | Issued-at, seconds since epoch |
| `nbf` | `1620345605` | Not-before; **optional** |
| `exp` | `1620345725` | Expiry, seconds since epoch |
| `aud` | `b4b-payments` | Audience; must be this value so tokens are decoded for B4B Payments |

**Token lifetime:** sources state that `exp` is required but do **not** document a mandated maximum/minimum lifetime. The example values imply a short window (~120 seconds between `iat` and `exp`); treat that as an illustration only unless a separate operational policy is confirmed.

### Request header

| Header | Example | Notes |
|---|---|---|
| `Authorization` | `Bearer {SIGNED_JWT}` | Signed JWT preceded by `Bearer` |
| `Content-Type` | `application/json` | Assumed on JSON bodies in the payments guide |

Example (redacted):

```http
Authorization: Bearer {SIGNED_JWT}
Content-Type: application/json
```

Bearer scheme is also reflected in the Oversight OpenAPI `bearerAuth` security scheme (JWT / RS512, same key-exchange description).

## Contrast: on-platform web services (`setting-up-authentication.md`)

`setting-up-authentication.md` describes the **separate** B4B web services surface (`/services/json/...`), **not** Oversight JWT auth:

| Topic | On-platform (documented) |
|---|---|
| Transport | HTTPS only |
| Auth | HTTP Basic Authentication (`Authorization: Basic {BASE64_CREDENTIALS}`) |
| Mandatory headers | `Authorization`, `Content-Type: application/json` |
| IP | Whitelisted client IPs |
| Environments | Staging `https://staging.b4bpayments.com`; Production `https://b4bpayments.com` |
| Multi-company | Institution-level credentials may require a `uuid` company parameter |

Do **not** use Basic Auth credentials against Oversight `/oversight/v1`, and do not assume Oversight JWT works on `/services/json`.
