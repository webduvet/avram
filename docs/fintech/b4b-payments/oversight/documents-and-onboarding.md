# Documents and company onboarding

Source: `setting-up-a-company.md` (plus onboarding mentions in beneficiaries/payments guide).

Setup is a one-time onboarding sequence per company. Oversight holds the regulatory company/ownership record; vIBANs link that record to Banking Circle-issued accounts.

Some sub-object endpoints (addresses, people, extended profile, uploads, vIBAN management) may require client provisioning. `POST /companies` and payment endpoints are available regardless.

Base URL (sandbox examples): `https://sandbox.b4bpayments.com/oversight/v1`. Auth: Bearer JWT — see [authentication.md](authentication.md).

## Recommended sequence

1. `POST /companies` — legal name, `external_ref`, registered office address, optional inline vIBANs  
2. `POST /companies/{company_id}/addresses` — additional addresses if needed  
3. `POST /companies/{company_id}/people` — contacts / shareholders / key individuals  
4. Document upload — `POST /companies/{company_id}/uploads` then `PUT` to signed storage URL  
5. `POST /companies/{company_id}/extended` — comprehensive KYC profile (once per company)  
6. `POST /companies/{company_id}/vibans` — register BC-issued vIBANs as needed  

Steps 2–5 support regulatory onboarding. They **do not gate payments** inside Oversight: a company with a vIBAN and an approved beneficiary can transact without an extended profile. Person sanctions outcomes also do not block payments.

## Create company

`POST /companies`

| Field | Type | Required | Notes |
|---|---|---|---|
| `legal_name` | string | yes | Full legal name |
| `external_ref` | string | yes | Unique within client |
| `trading_name` | string | no | |
| `address` | object | yes (recommended) | Registered office (inline) |
| `vibans` | array | no | Optional inline `{ number, currency }` |

### Example request / response (placeholders)

```json
{
  "legal_name": "{COMPANY_LEGAL_NAME}",
  "external_ref": "{YOUR_COMPANY_REF}",
  "trading_name": "{TRADING_NAME}",
  "address": {
    "line_1": "{LINE_1}",
    "line_2": "{LINE_2}",
    "city": "{CITY}",
    "region": "{REGION}",
    "postcode": "{POSTCODE}",
    "country": "GB"
  },
  "vibans": [
    { "number": "{VIBAN}", "currency": "GBP" }
  ]
}
```

```json
{
  "id": "{company_id}",
  "legal_name": "{COMPANY_LEGAL_NAME}",
  "trading_name": "{TRADING_NAME}",
  "external_ref": "{YOUR_COMPANY_REF}",
  "address": {
    "line_1": "{LINE_1}",
    "line_2": "{LINE_2}",
    "city": "{CITY}",
    "region": "{REGION}",
    "postcode": "{POSTCODE}",
    "country": "GB"
  },
  "vibans": [
    { "number": "{VIBAN}", "currency": "GBP" }
  ]
}
```

Reuse the returned address UUID from company create as the registered office; do not recreate it via `POST /addresses`.

## Addresses

`POST /companies/{company_id}/addresses`

| Field | Required | Notes |
|---|---|---|
| `type` | yes | `cardholder`, `business`, `registered_office`, `trading`, `ubo` (default `registered_office`) |
| `line_1` | yes | |
| `line_2` / `line_3` | no | |
| `city` | yes | |
| `region` / `postcode` | no | |
| `country` | yes | ISO 3166-1 alpha-2 |

## People

`POST /companies/{company_id}/people`

Roles: `contact`, `key_individual`, `shareholder`. Types: `natural_person` or `legal_entity`. Key individuals require `address_id`. Natural persons need identity document fields (`document_type`, `document_number`, `document_country`). `callback_url` required for sanctions status POSTs (informational).

Example (placeholders; no personal names retained):

```json
{
  "type": "natural_person",
  "role": "key_individual",
  "first_name": "{FIRST_NAME}",
  "last_name": "{LAST_NAME}",
  "email": "{EMAIL}",
  "telephone_number": "{PHONE}",
  "date_of_birth": "{YYYY-MM-DD}",
  "country_of_birth": "GB",
  "countries_of_citizenship": ["GB"],
  "positions_within_business": ["Director"],
  "document_type": "passport",
  "document_number": "{DOCUMENT_NUMBER}",
  "document_country": "GB",
  "address_id": "{address_id}",
  "callback_url": "https://example.com/webhooks/b4b/people"
}
```

`document_type` enum: `passport`, `identity_card`, `driving_license`, `residence_permit`.

## Document upload (two-step direct-to-storage)

File bytes never pass through Oversight. Repeat per file; collect signed IDs for the extended profile.

### Step 1 — mint upload slot

`POST /companies/{company_id}/uploads`

| Field | Required | Notes |
|---|---|---|
| `filename` | yes | Including extension |
| `byte_size` | yes | Positive integer; enforced on PUT |
| `checksum` | yes | **Base64-encoded MD5** of file content (not hex) |
| `content_type` | yes | e.g. `application/pdf` |

```json
{
  "filename": "business-registry-extract.pdf",
  "byte_size": 245760,
  "checksum": "{BASE64_MD5}",
  "content_type": "application/pdf"
}
```

Response (`201`):

```json
{
  "id": "{SIGNED_UPLOAD_TOKEN}",
  "url": "https://{STORAGE_HOST}/companies/{company_id}/...?...&X-Amz-Expires=300&...",
  "headers": {
    "Content-Type": "application/pdf",
    "Content-MD5": "{BASE64_MD5}",
    "Content-Disposition": "inline; filename=\"business-registry-extract.pdf\""
  }
}
```

- `id` is a **signed token** (not a UUID) — store and return verbatim in the profile `documents` array  
- `url` is time-limited; complete PUT promptly  
- `headers` must be sent **exactly** on the PUT  

### Step 2 — PUT file to storage

```http
PUT {url_from_step_1}
Content-Type: application/pdf
Content-MD5: {BASE64_MD5}
Content-Disposition: inline; filename="business-registry-extract.pdf"

<binary file content>
```

Success: `200 OK` empty body. Size/checksum mismatch → `400`. Expired URL → `403`.

### Reference in extended profile

```json
"documents": [
  { "type": "business_registry_extract", "document": "{SIGNED_UPLOAD_TOKEN}" },
  { "type": "articles_of_association", "document": "{SIGNED_UPLOAD_TOKEN_2}" }
]
```

Errors if PUT not completed: `{ "errors": { "documents": ["must be uploaded first"] } }`. Bad token: `{ "errors": { "documents": ["must exist"] } }`.

## Extended company profile

`POST /companies/{company_id}/extended` — **once** per company (no integrator update endpoint). Requires addresses/people/uploads already created.

Mandatory highlights:

- `company_type` enum (`limited`, `limited_liability_company`, …)  
- `nace_codes` — NACE Rev. 2.1 level-four `NN.NN` (not NAICS)  
- `documents` — **`business_registry_extract` always required**  
- Document `type` enum: `business_registry_extract`, `idv_check`, `psu_bank_statement`, `articles_of_association`, `tax_declaration`, `contracts_agreements`, `invoice`, `psu_residence_permit`, `ubo_registry_extract`, `ownership_structure`, `other`

Sub-objects (summary): `compliance`, `financial` (`product_types`: `card` and/or `payment`), conditional `cards` / `payments`, `ownership_structure` (shareholders / key_individuals with optional PEP associations), top-level `contacts`.

Invalid NACE:

```json
{ "errors": { "nace_codes": ["is invalid"] } }
```

Missing registry extract:

```json
{ "errors": { "documents": ["business registry extract is required"] } }
```

## vIBANs

`POST /companies/{company_id}/vibans`

```json
{ "number": "{VIBAN}", "currency": "EUR" }
```

Also documented: `GET /companies/{company_id}/vibans`, `POST .../vibans/{number}/archive`, `POST .../vibans/{number}/unarchive`. Only register vIBANs BC actually issued for that company/currency; Oversight authorises debit against this registration.

## Hard rules (server-enforced)

- Company must exist before sub-objects (`404` if wrong/`company_id`)  
- Key individual Person requires `address_id`  
- Documents must be fully PUT before extended profile  
- Person UUIDs on profile must belong to the same company  
- Extended profile create-once  
- `business_registry_extract` required  
- NACE must be valid level-four NACE 2.1  

`external_ref` uniqueness on Companies (and Beneficiaries/Payments) supports idempotent create retries.

## Extended profile example (redacted)

`POST /companies/{company_id}/extended` — payments-only UK limited company shape from the source worked example:

```json
{
  "company_type": "limited",
  "company_number": "{COMPANY_NUMBER}",
  "company_website": "https://example.com",
  "registered_office_address_id": "{registered_office_address_id}",
  "business_address_id": "{business_address_id}",
  "business_registry_url": "https://example.com/registry/{COMPANY_NUMBER}",
  "number_of_employees": 42,
  "nace_codes": ["46.51", "47.91"],
  "business_activities": "{DESCRIPTION}",
  "primary_business_operations_country": "GB",
  "countries_of_registration": ["GB"],
  "tax_residence_countries": ["GB"],
  "vat_number": "{VAT_NUMBER}",
  "compliance": {
    "risk_category": "medium",
    "passive_income": false,
    "passive_income_over_50_pct_sales": false,
    "pending_lawsuits": "None",
    "trust_trustee_agreement": false,
    "bearer_shares": false
  },
  "financial": {
    "product_types": ["payment"],
    "account_number": "{ACCOUNT_OR_IBAN}",
    "financial_institution": "SC112233"
  },
  "payments": {
    "purpose_of_incoming_payments": "{INCOMING_PURPOSE}",
    "purpose_of_outgoing_payments": "{OUTGOING_PURPOSE}",
    "incoming_currencies": ["GBP", "EUR"],
    "outgoing_currencies": ["GBP", "EUR", "USD"],
    "incoming_monthly_payment_volume": 500,
    "outgoing_monthly_payment_volume": 200,
    "incoming_monthly_payment_value": 250000,
    "outgoing_monthly_payment_value": 180000,
    "anticipated_beneficiaries": ["{BENEFICIARY_CLASS}"],
    "anticipated_payment_destinations": ["GB", "DE", "FR", "US"]
  },
  "ownership_structure": {
    "shareholders": [
      { "person_id": "{shareholder_person_id}", "share_percentage": 100, "nominee": false }
    ],
    "key_individuals": [
      {
        "person_id": "{director_person_id}",
        "roles": ["director", "ubo"],
        "share_percentage": 100,
        "pep_associations": []
      }
    ]
  },
  "contacts": [
    { "person_id": "{director_person_id}", "type": ["primary", "compliance"] }
  ],
  "documents": [
    { "type": "business_registry_extract", "document": "{SIGNED_UPLOAD_TOKEN}" },
    { "type": "articles_of_association", "document": "{SIGNED_UPLOAD_TOKEN_2}" },
    { "type": "ubo_registry_extract", "document": "{SIGNED_UPLOAD_TOKEN_3}" }
  ]
}
```

Product-type variants: `["payment"]` → include `payments` only; `["card"]` → include `cards` only; both → include both sub-objects.

## Updates

- `PATCH /companies/{company_id}` — same shape as create  
- `PATCH /companies/{company_id}/addresses/{address_id}` — address updates  
- Extended profile: create-once; revisions via account team (no client update endpoint)

## Person webhook payload (optional)

```json
{ "sanctions_status": "pass" }
```

Values: `pass` | `review` | `fail`. Unsigned; informational; does not gate payments.
