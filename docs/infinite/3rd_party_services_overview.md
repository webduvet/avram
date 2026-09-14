# Third Party Services used in Infinite Platform

Already covered (per this session's work): Worldline, B4B, Banking Circle, ACI-webhook, generic merchant-verification — via fintech-sim-lab, already running. Floci/LocalStack covers S3/SQS/SNS/EventBridge. 2× Postgres + Redis are self-hosted infra.

Everything else defaults to a true no-op — PostHog, Datadog (APM+RUM), SonarCloud, AWS SES (mailer's NODE_ENV=development console-logs instead), all 8 validation-service vendors, LexisNexis, IBAN.com, flagcdn/Wikimedia/ipapi.co/MUI-X-license. Copy .env.example
verbatim and these apps boot and run.

## Four actual blockers if you touch these specific apps:

┌────────────────────────────┬──────────────────────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────┐
│ App                        │ Blocker                                                                      │ Fix                                                                              │
├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────┤
│ apps/smser                 │ Crashes at boot unless Twilio or TextMagic creds present                     │ .env.example placeholders satisfy it — real send fails at request time, harmless │
├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────┤
│ apps/document              │ Crashes at boot without SignNow creds (whole app, incl. unrelated PDF-merge) │ Same — placeholders satisfy boot                                                 │
├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────┤
│ apps/gateway login         │ CaptchaFox enabled by default, fails closed → every non-OTP login 401s       │ Set CAPTCHAFOX_ENABLED=false                                                     │
├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────┤
│ apps/pay-by-link-simulator │ ACI checkout creds have zero graceful fallback, throws per-request           │ Only matters if you run this app                                                 │
└────────────────────────────┴──────────────────────────────────────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────┘

────────────────────────────────────────────────────────────────────────────────

## 1. Core infrastructure (self-hosted, not SaaS)

┌────────────────────────┬─────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Service                │ Used by                                                 │ Can run without it?                                                                             │
├────────────────────────┼─────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Postgres — platform DB │ every backend app via DATABASE_URL                      │ No — hard dependency everywhere                                                                 │
├────────────────────────┼─────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Postgres — settle DB   │ settle-* apps, apps/webhooks/banking-circle, @settle/db │ No — hard dependency for settle domain                                                          │
├────────────────────────┼─────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Redis                  │ @lightyear/cache (rate-limiting, throttler storage)     │ Yes — RedisCacheModule falls back to in-memory cache when REDIS_ENABLED≠'1' or creds incomplete │
├────────────────────────┼─────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Floci/LocalStack       │ S3, SQS, SNS, EventBridge (see below)                   │ It is the "run without real AWS" answer                                                         │
└────────────────────────┴─────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────┘

### 2. AWS managed services

┌──────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────────┐
│ Service          │ Used by                                                                                                     │ Required or optional                                 │ Can run without it?                                                           │
├──────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────┤
│ S3               │ gateway (LIFECYCLE_S3_BUCKET, UPLOAD_BUCKET), document (shared lifecycle bucket),                           │ Bucket name required by schema in gateway/document   │ Needs AWS_S3_ENDPOINT=http://localhost:4566 (LocalStack) or real AWS creds —  │
│                  │ settle-reporting/billing-statements/commissions (SETTLEMENT_S3_BUCKET), settle-ingest/calc                  │ (boot-time string check only)                        │ no in-code no-op for actual reads/writes                                      │
├──────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────┤
│ SQS              │ gateway (5 internal consumer queues), document→verification-service handoff, settle pipeline (already       │ Optional — SQS_ENABLED flag, default false           │ Yes — consumers just log "disabled" and never poll                            │
│                  │ covered)                                                                                                    │                                                      │                                                                               │
├──────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────┤
│ SNS              │ shared messaging lib                                                                                        │ Optional — SNS_ENABLED flag                          │ Yes                                                                           │
├──────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────┤
│ EventBridge      │ gateway's own internal domain-event bus (auth.login, boarding.completed, countries.updated, ...) + settle   │ Optional — EVENTBRIDGE_ENABLED flag, default false   │ Yes — putEvents() short-circuits to a dry-run success, debug-log only         │
│                  │ pipeline                                                                                                    │ in root .env.example                                 │                                                                               │
├──────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────┤
│ SES              │ apps/mailer — real SESv2Client.SendEmailCommand, not vestigial                                              │ Only generic sender/domain fields required, not AWS  │ Yes — NODE_ENV=development makes sendEmail() console-log and return a fake    │
│                  │                                                                                                             │ creds                                                │ ID; SES never touches the network                                             │
├──────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────┤
│ SSM/Secrets      │ libs/shared/secrets.loadSecrets(), used by Lambda-style apps only (not auth/banking-circle/dal, which read  │ N/A                                                  │ Yes — no-op whenever SECRETS_PATH is unset (always true locally)              │
│ Manager          │ process.env directly)                                                                                       │                                                      │                                                                               │
└──────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────┴──────────────────────────────────────────────────────┴───────────────────────────────────────────────────────────────────────────────┘

## 3. Payment / card / acquiring vendors

┌─────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────┬────────────────────────────────────────────────────────────────┐
│ Vendor                                                  │ Used by                                                                  │ Required?                                     │ Local mock                                                     │
├─────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────────────────────┤
│ Worldline (acquirer, SFTP+PGP settlement files)         │ acquirer-fts, gateway's downloader cron                                  │ Real files needed for that flow               │ ✅ fintech-sim-lab (:2222)                                     │
├─────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────────────────────┤
│ B4B Payments (payout rail + sanctions/PEP screening)    │ accounts-settlement, gateway's B4B boarding modules                      │ Real payout flow needs it                     │ ✅ fintech-sim-lab (:8086)                                     │
├─────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────────────────────┤
│ Banking Circle (safeguarding account, webhooks)         │ settle-processing (SGA balance check), apps/webhooks/apps/banking-circle │ Gates MERCHANT_SETTLEMENT/INFINITE_SETTLEMENT │ ✅ fintech-sim-lab (:8085/:8095)                               │
├─────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────────────────────┤
│ ACI — webhook (card-payment notification)               │ settle-aci-webhook, proxied by gateway                                   │ Optional per-notification                     │ ✅ fintech-sim-lab (:8087), same AES-256-GCM secret both sides │
├─────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────────────────────┤
│ ACI — card gateway / 3DS (AciGatewayService)            │ gateway (merchant boarding, BIN config)                                  │ Required at gateway boot (schema, no default) │ Placeholder values satisfy boot; not mocked anywhere           │
├─────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────────────────────┤
│ ACI — hosted Checkout / Pay-by-Link (eu-test.oppwa.com) │ apps/pay-by-link-simulator only                                          │ Required, zero fallback, throws per-request   │ Not mocked; only relevant if you run this one app              │
└─────────────────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────────────┴───────────────────────────────────────────────┴────────────────────────────────────────────────────────────────┘

Three genuinely distinct ACI integration surfaces — easy to conflate.

## 4. KYC / AML / compliance / business-data vendors

All isolated to apps/verification-service and apps/validation-service (two separate apps, both reached via apps/gateway proxies — gateway itself never calls any of these directly).

┌───────────────────┬─────────────────────────────────────────────────┬────────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────┐
│ Vendor            │ App                                             │ Purpose                                        │ Mock-by-default?                                                 │ Can run without it?                                                         │
├───────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ Acuris "KYC6"     │ verification-service                            │ Sanctions + PEP screening, ongoing monitoring  │ ✅ yes, KYC6_WORKAROUND_FORCE=true in .env.example               │ Fully mocked out of the box                                                 │
├───────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ Creditsafe        │ verification-service and validation-service     │ Business credit report, UK bank verification,  │ No (verification-service); yes via CREDITSAFE_WORKAROUND_FORCE   │ verification-service: boots, real calls fail at request time.               │
│                   │ (separate creds)                                │ monitoring                                     │ (validation-service)                                             │ validation-service: graceful {valid:false}                                  │
├───────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ LexisNexis        │ verification-service                            │ Enhanced due diligence / identity verification │ No                                                               │ Boots (placeholder satisfies schema), real calls fail at request time       │
│ (IDU/IVI)         │                                                 │                                                │                                                                  │                                                                             │
├───────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ IBAN.com          │ verification-service and validation-service     │ IBAN/BIC/bank-account-name verification        │ No (WORKAROUND_FORCE is dead code in both)                       │ Boots; graceful {valid:false} in validation-service, real-call-failure in   │
│                   │                                                 │                                                │                                                                  │ verification-service                                                        │
├───────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ HMRC              │ validation-service                              │ UK VAT number lookup (OAuth2)                  │ Via HMRC_WORKAROUND_FORCE                                        │ Yes, fully graceful                                                         │
├───────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ Companies House   │ validation-service                              │ UK company registry                            │ Via *_WORKAROUND_FORCE                                           │ Yes, inline mock data                                                       │
├───────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ Vatlayer          │ validation-service                              │ EU VAT validation                              │ No forced-mock, but soft-fails                                   │ Yes, {valid:false}                                                          │
├───────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ HitHorizons       │ validation-service                              │ Non-UK EU company/VAT verification (9          │ Via *_WORKAROUND_FORCE                                           │ Yes, inline mock data                                                       │
│                   │                                                 │ countries)                                     │                                                                  │                                                                             │
├───────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ Loqate (GBG)      │ validation-service                              │ Address/phone/email/UK bank validation         │ Via *_WORKAROUND_FORCE, extra APP_ENV gate for bank data         │ Yes                                                                         │
├───────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ FinCodes          │ validation-service                              │ SEPA IBAN/BIC validation                       │ Soft-fails only                                                  │ Yes, returns null/{success:false}                                           │
└───────────────────┴─────────────────────────────────────────────────┴────────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────┘

fintech-sim-lab's generic "verify" stub fronts verification-service's own contract, not these vendors individually — pointing gateway at it skips all four verification-service vendors at once, but does nothing for validation-service (which doesn't need it — it
already self-degrades).

## 5. Communications (SMS / email / e-signature)

┌───────────┬──────────────────────────────────────────────────────────────┬───────────────────────────────┬────────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────┐
│ Vendor    │ App                                                          │ Role                          │ Required?                                                      │ Can run without it?                                                         │
├───────────┼──────────────────────────────────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ Twilio    │ smser                                                        │ Primary SMS                   │ Conditionally — needs Twilio or TextMagic to boot              │ Placeholders satisfy boot; per-call failure auto-falls-through to TextMagic │
├───────────┼──────────────────────────────────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ TextMagic │ smser                                                        │ Fallback SMS                  │ Same condition                                                 │ Same                                                                        │
├───────────┼──────────────────────────────────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ AWS SES   │ mailer                                                       │ Sole email transport          │ See §2                                                         │ Yes in dev mode                                                             │
├───────────┼──────────────────────────────────────────────────────────────┼───────────────────────────────┼────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ SignNow   │ document (real caller); gateway (pure proxy, no direct call) │ Merchant contract e-signature │ Required at document boot, whole app incl. unrelated PDF-merge │ Placeholders satisfy boot; real signing fails at request time               │
└───────────┴──────────────────────────────────────────────────────────────┴───────────────────────────────┴────────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────┘

## 6. Security, observability & frontend misc

┌─────────────────────────┬────────────────────────────────────────────────────────────────┬──────────────────────────────────┬───────────────────────────────────────────────────────────────────┐
│ Vendor                  │ Used by                                                        │ Required?                        │ Can run without it?                                               │
├─────────────────────────┼────────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ CaptchaFox              │ gateway login, merchant-portal login/register                  │ Enabled by default, fails closed │ Set CAPTCHAFOX_ENABLED=false (+ NEXT_PUBLIC_ variant)             │
├─────────────────────────┼────────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ PostHog (backend)       │ acquirer-fts, gateway, settle-processing, verification-service │ No                               │ Yes — every call site is if (!client) return                      │
├─────────────────────────┼────────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ PostHog (frontend)      │ lightyear                                                      │ No                               │ Yes — gated behind cookie-consent opt-in, off by default          │
├─────────────────────────┼────────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ Datadog dd-trace (APM)  │ 11 apps via shared tracer.ts                                   │ No                               │ Yes — vendor SDK itself no-ops on DD_TRACE_ENABLED=false/no agent │
├─────────────────────────┼────────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ Datadog Browser RUM     │ lightyear only                                                 │ No                               │ Yes — NEXT_PUBLIC_DATADOG_RUM_ENABLED default false               │
├─────────────────────────┼────────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ SonarCloud              │ dev/CI tooling only                                            │ N/A                              │ Yes — never imported by runtime code                              │
├─────────────────────────┼────────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ flagcdn.com / Wikimedia │ lightyear (flag/logo icons)                                    │ No                               │ Yes — identical hardcoded fallback                                │
├─────────────────────────┼────────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ ipapi.co                │ merchant-portal + lightyear (PhoneNumberInput)                 │ No                               │ Yes — falls back to a constant country                            │
├─────────────────────────┼────────────────────────────────────────────────────────────────┼──────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ MUI X license key       │ lightyear                                                      │ No                               │ Yes — unlicensed watermark only                                   │
└─────────────────────────┴────────────────────────────────────────────────────────────────┴──────────────────────────────────┴───────────────────────────────────────────────────────────────────┘

Not present anywhere: an external IdP/OAuth/SSO provider. Auth is fully self-hosted (bcrypt + JWT + otplib TOTP) end-to-end; jose/jwks-rsa are present but only decode/are-unused, never verify a third-party token.

