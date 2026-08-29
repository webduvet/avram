# AWS temporary elevated access: request and approval via Microsoft Teams

**Scope:** Multi-account AWS Organization using IAM Identity Center. Target pattern: a requester asks in Microsoft Teams for a named permission set (for example `first_responder`) on specific accounts or an OU, with a reason and duration; an approver acts in Teams; AWS then grants access for that window and revokes it automatically.

**As of:** 29 August 2026.

**Bottom line:** There is no AWS-native product that implements request *and* approval inside Microsoft Teams and then time-boxes IAM Identity Center permission-set assignments. AWS TEAM is the official sample for Identity Center JIT, but its UX is a web app; native chat is Slack notifications plus SNS, not Teams. Amazon Q Developer in chat applications (formerly AWS Chatbot) can talk to Teams but **cannot** grant Identity Center permission sets (`sso:*` and `identitystore:*` are hard-denied). Entra PIM for Groups can time-box **group membership** that SCIM then syncs into standing Identity Center group assignments; it does not call Identity Center assignment APIs, and approval lives in Entra, not Teams. For the stated Teams-native request/approve UX, a custom bot plus Step Functions that call Identity Center APIs is the path that actually matches.

---

## 1. Ranked options

Ranked for a team that already uses Microsoft Teams and AWS, against the target pattern (request in Teams, approve in Teams, automatic grant/revoke in Identity Center).

| Rank | Option | Teams request/approve | AWS grant/revoke | Fit |
| --- | --- | --- | --- | --- |
| 1 | Custom Teams bot + Adaptive Cards + Step Functions + Identity Center APIs | Yes | Direct `CreateAccountAssignment` / `DeleteAccountAssignment` | Matches the target UX. Highest build and ownership cost. |
| 2 | AWS TEAM (web portal) + SNS/SES into Teams | No (request/approve in TEAM UI; Teams is notify-only unless you fork TEAM) | Proven Step Functions grant/revoke | Fastest AWS-blessed engine. Partial UX fit. |
| 3 | Entra PIM for Groups + SCIM into Identity Center | No (activate/approve in My Access / Entra; optional Entra notifications) | Indirect: standing **group** assignments; membership appears/disappears via SCIM | Strong if Entra is already the IdP and elevation can be **group-shaped**, not “this account + this permission set in chat”. |
| 4 | Amazon Q Developer in chat applications (Teams) | Not as a PAM workflow | **Cannot** assign Identity Center permission sets | Use for CloudWatch/Support/SSM, not privilege elevation. |
| 5 | Commercial PAM (Teleport, CyberArk, StrongDM, Apono, Okta Access Requests, Tenable) | Varies (Teleport has a Teams plugin) | Partner-managed assignments or proxy | Buy instead of build when you need multi-cloud PAM, session recording, or a supported vendor. |

### 1. Recommended — custom Teams workflow (AWS-native grant path)

**What it is.** A Teams bot (Azure Bot Service) posts Adaptive Cards for request and approve/deny. An AWS API Gateway / Lambda (or EventBridge Pipe) receives the decision, a Step Functions state machine enforces eligibility and two-person rule, then calls Identity Center `CreateAccountAssignment`. EventBridge Scheduler (or a Step Functions Wait) later calls `DeleteAccountAssignment`. DynamoDB holds request state keyed by a correlation `requestId` that is also written into Teams card text and CloudTrail custom context.

**Why first.** It is the only option that implements the stated channel/bot UX without waiting for a product that does not exist. Grant/revoke uses the same Identity Center APIs TEAM uses. You control approver groups, prod two-person rule, OU expansion, and expiry.

**Tradeoffs.** You own eligibility policy, identity mapping (Teams AAD object id / UPN → Identity Center `UserId`), card refresh, retries, and security of the bot secret. Identity Center assignments are **per account**, not per OU — OU requests must enumerate member accounts. Assignment deletion does **not** kill already-issued console/CLI sessions; permission-set session duration still applies. This is sample-grade architecture unless you treat it as a production control plane (WAF, least privilege, break-glass independent of the bot).

### 2. AWS TEAM + Teams as notification surface

**What it is.** Official AWS Samples solution: React SPA on Amplify, Cognito + SAML via Identity Center, AppSync, DynamoDB, Step Functions. Requesters and approvers work in a web UI launched from the Identity Center portal. Grant workflow creates a temporary account assignment; revoke deletes it. Eligibility and approval policies can name accounts **or OUs** (OU means accounts directly under that OU, not nested OUs). Native notifications: Amazon SES, Slack (bot OAuth token), Amazon SNS topic `TeamNotifications-main`. **There is no native Microsoft Teams bot, channel, or Azure app registration in TEAM.**

**Setup that exists today.** Deploy TEAM in the Identity Center **delegated administrator** account (not the management account). Enable SNS in TEAM Settings, subscribe a Lambda (or EventBridge API Destination) that posts into Teams (Workflows webhook or Graph). Approvers still click through to the TEAM URL.

**Tradeoffs.** Fastest path to a tested grant/revoke engine, CloudTrail Lake session viewer, and eligibility/approval policies. Does **not** satisfy “approve in Teams” without a custom frontend against TEAM’s AppSync API (undocumented, version-fragile) or a fork. Slack is first-class; Teams is DIY on SNS. TEAM cannot assign permission sets on the **management account** when deployed in a delegated admin account. Patch to a current release (a 2025 authorization bypass, CVE-2025-1969, was addressed in 1.2.2). Amplify/CodeCommit-era deploy scripts need operational care.

**When to pick this.** Teams-native approval can wait; you need JIT on Identity Center this quarter; auditors want a single web console.

### 3. Entra PIM for Groups + SCIM (federation already in place)

**What actually works.** If Entra ID is the SAML IdP for Identity Center and SCIM provisioning is on:

1. Create an Entra **security** group per privilege bundle (for example `AWS-first_responder-prod`).
2. Enable **PIM for Groups** on that group; make users **eligible** members.
3. Assign the group to the Identity Center enterprise app so SCIM provisions the group and, on activation, its membership.
4. In Identity Center, create a **standing** `GROUP` assignment of that group to the relevant permission set(s) and account(s).

Activation adds the user to the Entra group; SCIM PATCHes membership into Identity Center; the user then sees the permission set in the AWS access portal. PIM settings can require MFA, justification, approval, and a maximum duration. This is documented by both Microsoft (PIM for Groups + app provisioning) and AWS (Security Blog, June 2025).

**What does not work / is Azure-only.**

- PIM for **Entra roles** and PIM for **Azure resources** do not grant AWS. Only **PIM for Groups** plus SCIM is the AWS path.
- PIM does **not** call `CreateAccountAssignment`. Account↔permission-set mapping is standing and group-based. There is no chat-time picker of “account 1234 + `first_responder` for 4 hours” unless you pre-create a group (and Identity Center assignment) per combination.
- Approval is in **My Access / Entra admin center**, not a Teams Adaptive Card. Do not treat Entra’s optional email/Teams *notifications* as a Teams approval workflow.
- Dynamic groups and groups synced from on-prem AD **cannot** be PIM-enabled. Nested group members **do not** SCIM to Identity Center (direct members only).
- PIM activation → SCIM is typically 2–10 minutes; if more than five activations hit the same enterprise app in 10 seconds, overflow waits for the ~40 minute incremental cycle. **Deactivation is incremental-cycle, not on-demand** (Microsoft: “Deactivation is done during the regular incremental cycle. It isn't processed immediately through on-demand provisioning.”).
- Active AWS sessions continue until the **permission set session duration** ends, even after PIM expiry and even after SCIM removal. AWS documented a correction (19 June 2025) that the realistic access window is PIM duration + SCIM lag + session duration (on the order of hours, not “exactly 1 hour”).
- Licensing: Entra ID P2 or Entra ID Governance for PIM for Groups.

**When to pick this.** The organization already lives in PIM, elevation can be expressed as a small set of privileged groups, and a few minutes of grant lag plus lingering sessions are acceptable. Poor fit for incident “this one account, now, approved in the war-room channel.”

### 4. Amazon Q Developer in chat applications — what it can and cannot do

**Can (Teams, standard/public channels only):**

- Deliver SNS-backed notifications (CloudWatch, EventBridge, AWS Support, Security Hub, etc.).
- Run many AWS CLI commands from the channel, subject to channel role ∩ user role ∩ **guardrail** policies.
- Custom action buttons that run CLI, Lambda, or SSM Automation using the **channel configuration’s IAM role**.
- Chat with Amazon Q (natural language) if `AmazonQDeveloperAccess` (or similar) is attached.

**Cannot:**

- **Grant or revoke IAM Identity Center permission-set assignments.** The service hard-denies CLI operations including `sso:*` and `identitystore:*` (see Non-supported operations). The IAM action for `CreateAccountAssignment` is `sso:CreateAccountAssignment`. Amazon Q’s own `q:CreateAssignment` / `q:DeleteAssignment` are for **Q Developer profiles**, not Identity Center.
- Operate in **Microsoft Teams private channels** (documented limitation).
- Provide an approval state machine, eligibility policy, or two-person rule for privilege elevation.

A custom-action button that **invokes a Lambda you wrote**, where that Lambda (not Chatbot) holds `sso:CreateAccountAssignment`, is not “Chatbot granting Identity Center access.” It is a thin, dangerous UI over a custom workflow: everyone in a standard channel who can press the button shares the channel role. Do not use that as PAM.

**Setup (for notifications/ops only):** Teams admin approves the Amazon Q app; in the Amazon Q Developer in chat applications console, register tenant + channel (Team ID, Tenant ID, Channel URL); attach a least-privilege channel role and guardrails; subscribe SNS topics. CloudFormation resource: `AWS::Chatbot::MicrosoftTeamsChannelConfiguration`.

### 5. Commercial options (short)

**Teleport.** Identity Center integration creates/deletes **user** account assignments on Access Request expiry; long-term access via SCIM’d groups. Native **Microsoft Teams plugin** (Azure Bot) notifies and links to approve/deny. Takes ownership of Identity Center users/groups/assignments when enabled — migration risk. Enterprise product; Teams is a plugin, not the control plane.

**CyberArk Secure Cloud Access.** On AWS’s **validated** Identity Center temporary-elevated-access partner list. Vault/session-centric PAM; JIT into AWS accounts/permission sets with audit. Teams is typically notification or a CyberArk add-on, not Adaptive Cards you design. Fits regulated orgs that already run CyberArk.

**StrongDM.** Proxy/Zero Trust to **data-plane** resources (Redshift, EC2, EKS) more than Identity Center permission-set assignment. JIT approvals exist; AWS console/Identity Center is not the core model. Useful if the pain is database/SSH/K8s access rather than console roles.

**Apono / Okta Access Requests / Tenable (Ermetic).** Also on AWS’s validated Identity Center TEAM partner list (docs: Temporary elevated access for AWS accounts). Okta Access Requests is natural if Okta is the IdP (this brief assumes Teams + Entra). Apono and Tenable emphasize entitlement graphs and cloud JIT. Evaluate against Identity Center assignment semantics, not only marketing “JIT.”

**AWS IAM Identity Center + third-party PAM generally.** Identity Center remains the account assignment plane; the vendor either (a) calls the same `CreateAccountAssignment` APIs TEAM uses, or (b) sits in front as a proxy and never assigns Identity Center roles. Prefer (a) if the AWS access portal / CLI SSO must keep working. Always confirm session revocation behavior.

---

## 2. Recommended path — concrete setup

This is option 1: Teams-native request/approve, AWS-native grant/revoke. Deploy the AWS side in the Identity Center **delegated administrator** account.

### 2.1 Microsoft Entra / Azure objects

1. **App registration** (single-tenant) for the bot. Record Application (client) ID, Directory (tenant) ID, client secret or (preferred) federated credentials.
2. **Azure Bot resource** (Azure Bot Service) using that app ID. Enable the **Microsoft Teams** channel. Messaging endpoint: `https://<api>/api/messages` (API Gateway custom domain or Bot Service with an AWS adapter; many designs put a small Azure Bot / Container Apps proxy that forwards to API Gateway, or host the bot on AWS with the Bot Framework SDK).
3. **Teams app manifest** (sideload in a test team, then publish to the org catalog):
   - Bot with scopes `personal`, `team` (avoid relying on private channels).
   - Adaptive Cards 1.4+ with `Action.Execute` (verb: `approve` / `deny` / `submitRequest`) and `fallback` `Action.Submit`.
   - Optional RSC: `ChannelMessage.Read.Group` if the bot must see requests without @mention; otherwise require `@Bot elevate ...` in a dedicated channel.
4. **Teams admin:** allow the app, pin it to the elevation team, restrict who can install it.
5. **Security groups in Entra** (synced via SCIM to Identity Center):
   - `aws-elevate-requesters`
   - `aws-elevate-approvers-nonprod`
   - `aws-elevate-approvers-prod` (no overlap with requesters for prod if you want team separation)
   - Map these to Identity Center groups; the bot authorizes against Identity Center group membership (or Entra group IDs in the token) — pick one source of truth.
6. **Graph / Bot permissions (keep minimal):**
   - Bot Framework connector (Teams channel) — this is how cards are posted/updated; **not** `ChannelMessage.Send` application Graph if you stay on Bot Framework.
   - If using Graph for proactive install or roster: `AppCatalog.Read.All` (admin), maybe `TeamsAppInstallation.ReadWriteForTeam.All` only if an installer job pushes the app. Prefer RSC at install time over tenant-wide Graph.
   - Do **not** grant `Directory.ReadWrite.All` or Group write. The bot must not be an Entra PIM actor unless you deliberately chose option 3.
7. **Identity mapping:** SAML `NameID` / SCIM `userName` should match the Teams UPN or mail so `identitystore:GetUserId` with `emails.value` or `userName` resolves the requester. Confirm this in the Entra Identity Center gallery app attribute mappings.

### 2.2 AWS accounts and services

| Piece | Where | Notes |
| --- | --- | --- |
| IAM Identity Center instance | Management or delegated admin (org instance) | Organization instance required for multi-account permission sets. |
| Workflow account | **Delegated administrator** for Identity Center | Same recommendation as TEAM. No other workloads. |
| Management account | Out of band | Delegated admin **cannot** create/delete assignments **on the management account**. Break-glass lives here, not in the bot. |
| Member accounts / OUs | Targets | Permission set (e.g. `first_responder`) provisioned to target accounts ahead of time (`ProvisionPermissionSet` if the set already exists). |

**Services in the workflow account:**

- Amazon API Gateway (HTTP API) + AWS WAF — Bot Framework messages and optional slash-command webhook.
- AWS Lambda — parse Adaptive Card `Action.Execute`, verify Bot Framework JWT (audience = bot app ID, issuer = Bot Framework / Entra), map user.
- AWS Step Functions — `Request → NotifyApprovers → WaitForDecision → (optional Wait until start) → Grant → Wait duration → Revoke`.
- Amazon DynamoDB — requests table (`requestId` PK, status, requester Identity Center UserId, permissionSetArn, accountIds[], duration, reason, approver, Teams `conversationId`/`activityId` for card updates).
- Amazon EventBridge Scheduler — one-time `at(...)` schedule per grant with `ActionAfterCompletion=DELETE` targeting the revoke Lambda. Preferred over DynamoDB TTL.
- IAM roles — see security caveats. Grant role allowed only listed permission-set ARNs and `sso:CreateAccountAssignment` / `sso:DeleteAccountAssignment` on the Identity Center instance; plus `identitystore:GetUserId`, `identitystore:DescribeUser`, `identitystore:IsMemberInGroups`; plus `organizations:ListAccountsForParent` / `ListChildren` for OU expansion.
- AWS KMS + Secrets Manager — bot app secret if not using federated creds; SNS/Teams webhook URLs.
- Amazon SNS / SES — expiry and denial notices (and a second channel if Teams is down).
- Organization CloudTrail (or CloudTrail Lake) — management events in the Identity Center home Region.
- Optional: Amazon Q Developer in chat applications on a **separate** ops channel for CloudWatch, **without** `sso:*` on its role.

### 2.3 Teams operational design

**Dedicated request channel vs bot DMs**

- **Dedicated standard channel** (for example `aws-elevation`) is the default: visible audit trail, @here for on-call, channel membership is a coarse ACL. Use a **standard** channel: Q Chatbot cannot use private channels; custom bots have limited private-channel support (cannot reliably post Adaptive Cards there).
- **Bot DMs** for the Adaptive Card to named approvers (user-specific views, max 60 users per card refresh list). Post a **redacted** summary in the channel (“request `req-…` pending, accounts …, set `first_responder`, 4h”) and the actionable card only to the approver group. That reduces rubber-stamp by random channel members.
- Do not use Incoming Webhooks as the approval mechanism (Office 365 connectors retired; Workflows webhooks are one-way unless you use Power Automate “Post card and wait for a response,” which still needs a robust identity check).

**Adaptive Cards**

- Request card: permission set dropdown (allow-listed names), account IDs and/or OU IDs, duration (enum: 1h/2h/4h/8h, cap in policy), ticket id, justification (required, min length), `requestId` already minted client-side or by the bot.
- Approver card: `Action.Execute` verbs `approve` / `deny`, `Input.Text` for deny/approve reason, `refresh.userIds` = approver MRIs so only they see buttons ([Universal Actions / user-specific views](https://learn.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/universal-actions-for-adaptive-cards/user-specific-views)).
- After decision, **update the original message** so the channel sees Approved/Denied + requestId (do not leave live Approve buttons).

**Who can approve**

- **Not** “anyone in the channel.” Authorize against a named Identity Center / Entra **approver group** for that account or OU (same model as TEAM approval policies).
- **Self-approval forbidden** (compare Identity Center UserId of requester vs actor). TEAM does this; copy it.
- **Prod:** two-person rule — two distinct approvers from `aws-elevate-approvers-prod`, or one approver plus a change ticket that the bot validates via ITSM API. Nonprod can be single approver.
- Channel membership is a **visibility** control, not an authorization control.

**Audit correlation**

Mint `requestId` (UUIDv4) at submit. Propagate it to:

- Teams card text and bot message `channelData`.
- DynamoDB item.
- Step Functions execution name / tag.
- EventBridge Scheduler name (`elevate-<requestId>-revoke`).
- Lambda `CreateAccountAssignment` — CloudTrail will record `userIdentity` of the **workflow role**, not the human. Put `requestId`, requester UserId, approver UserId, permission set, accounts, duration in:
  - DynamoDB (system of record for the human workflow),
  - the Teams thread,
  - a custom EventBridge event or CloudWatch log line with the same id,
  - optionally `tags` if you tag a supporting resource (assignments themselves are not taggable in a useful way).

Human session activity after grant is in CloudTrail in the **target account** with Identity Center user id (`userIdentity.identityStoreUserId` / `onBehalfOf` fields — see Identity Center CloudTrail docs). Join: `requestId` → UserId + time window → CloudTrail Lake query in TEAM style.

**Expiry / denial notifications**

- Deny: update card; DM requester; optional SES.
- Grant: DM requester “permission set X on account(s) Y until Z; sign in via AWS access portal.”
- Expiry: Scheduler invokes revoke; on `DescribeAccountAssignmentDeletionStatus` = SUCCEEDED, post to thread and DM. On failure, page the security channel (SNS → PagerDuty/Teams ops).
- Reminders: Scheduler 15 minutes before expiry.

**Break-glass if Teams is down**

Independent of Teams **and**, if possible, of Identity Center’s Region:

1. Pre-build **emergency IAM SAML federation** from Entra (or a second IdP) **directly into a small number of IAM roles** in a dedicated emergency account, as in [Set up emergency access to the AWS Management Console](https://docs.aws.amazon.com/singlesignon/latest/userguide/emergency-access.html). This still works if Identity Center is down but IAM + Entra are up.
2. If Entra is also down: a handful of IAM users with MFA, credentials in an **offline** vault (not Entra-conditional). See [Break-glass access](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/break-glass-access.html).
3. Document who may use it, dual-control to retrieve the vault, and a requirement to file a TEAM/custom request **after** the incident for audit.
4. Do not put break-glass behind the same bot, the same Lambda, or the same region-only Cognito/TEAM stack.

---

## 3. Identity Center APIs and assignment lifecycle

SDK/CLI namespace: `sso-admin`. IAM action prefix: `sso:`. Identity lookup: `identitystore`.

### 3.1 APIs involved

| API | Client | IAM action | Role |
| --- | --- | --- | --- |
| `CreateAccountAssignment` | `sso-admin` | `sso:CreateAccountAssignment` | Grant. Async; returns `RequestId` + `Status`. |
| `DescribeAccountAssignmentCreationStatus` | `sso-admin` | `sso:DescribeAccountAssignmentCreationStatus` | Poll until `SUCCEEDED` / `FAILED`. |
| `DeleteAccountAssignment` | `sso-admin` | `sso:DeleteAccountAssignment` | Revoke. Also async. |
| `DescribeAccountAssignmentDeletionStatus` | `sso-admin` | `sso:DescribeAccountAssignmentDeletionStatus` | Poll revoke. |
| `ListAccountAssignments` | `sso-admin` | `sso:ListAccountAssignments` | Idempotency / “already has standing access?” |
| `ListAccountAssignmentsForPrincipal` | `sso-admin` | `sso:ListAccountAssignmentsForPrincipal` | Requester’s current entitlements. |
| `ProvisionPermissionSet` | `sso-admin` | `sso:ProvisionPermissionSet` | If the set is new or policies changed; **not** needed on every grant if already provisioned. |
| `ListInstances` | `sso-admin` | `sso:ListInstances` | Resolve instance ARN / Identity Store id. |
| `GetUserId` | `identitystore` | `identitystore:GetUserId` | UPN/email → `UserId`. Paths: `userName`, `emails.value`. |
| `DescribeUser` / `IsMemberInGroups` | `identitystore` | corresponding `identitystore:*` | Eligibility / approver group check. |
| `ListAccountsForParent` / `ListChildren` / `ListAccounts` | `organizations` | `organizations:ListAccountsForParent` etc. | Expand OU → account IDs. |
| `ListPermissionSets` / `DescribePermissionSet` | `sso-admin` | `sso:ListPermissionSets` | Resolve name `first_responder` → ARN. |

**Hard constraint:** `TargetType` is only `AWS_ACCOUNT`. There is no `AWS_OU`. TEAM’s OU eligibility is UI sugar: it lists accounts **directly under** the OU and still calls per-account assignment APIs.

**Quota:** `CreateAccountAssignment` allows 15 outstanding async calls (not increasable). For an OU with many accounts, serialize or batch with backoff.

**PrincipalType:** Use `USER` for JIT (TEAM’s model). `GROUP` is for standing access (PIM model). Mixing “temporary user assignment” with SCIM-managed group assignments on the same permission set is fine; do not delete GROUP assignments from the JIT robot.

### 3.2 Lifecycle (custom or TEAM)

```
pending  --(approver)--> approved --(start time)--> granting --(Create SUCCEEDED)--> in_progress
    |                         |                                              |
    +-- rejected/expired      +-- cancelled                                  +-- (duration elapsed | revoke)
                                                                              --> revoking --(Delete SUCCEEDED)--> ended
```

1. **Create** `CreateAccountAssignment(InstanceArn, PermissionSetArn, PrincipalType=USER, PrincipalId, TargetType=AWS_ACCOUNT, TargetId=<account>)`.
2. Poll `DescribeAccountAssignmentCreationStatus` until `SUCCEEDED` (Identity Center provisions an IAM role in the target account, name typically `AWSReservedSSO_<permissionSet>_<hash>`).
3. Requester signs in via the **AWS access portal** (or `aws sso login`). Portal session duration and permission-set session duration are **independent** of the JIT window.
4. **Expire:** `DeleteAccountAssignment` with the same tuple. Poll deletion status.
5. **Sessions:** deleting the assignment hides the permission set on next portal load; **already issued STS credentials remain valid until their session duration**. Set permission-set session duration short (1 hour is TEAM’s guidance). For true kill, follow [How to revoke federated users’ active AWS sessions](https://aws.amazon.com/blogs/security/how-to-revoke-federated-users-active-aws-sessions/).

### 3.3 How to expire (pick one)

| Mechanism | Verdict |
| --- | --- |
| **EventBridge Scheduler** one-time `at(YYYY-MM-DDThh:mm:ss)` + `ActionAfterCompletion=DELETE`, target = revoke Lambda | **Preferred.** Exact time, retries, DLQ, schedule auto-deleted. |
| **Step Functions Wait** then invoke revoke (TEAM’s Grant state machine) | Fine up to 1 year Wait. Execution history cost; simpler than Scheduler for “duration hours from now.” |
| **DynamoDB TTL + stream → Lambda** | **Do not use as the sole revoke.** TTL deletion is **best-effort** and can lag many hours (documented up to 48 hours). Use TTL only to garbage-collect old request rows **after** revoke succeeded. |
| CloudWatch Events cron | Too coarse. |

Idempotency: revoke Lambda should `ListAccountAssignments` and no-op if already gone; cancel the Scheduler schedule if a human revokes early.

### 3.4 Eventing

Identity Center does not emit a rich native “assignment expired” EventBridge event. You get **CloudTrail API events** (`source: aws.sso`, `eventSource: sso.amazonaws.com`, `eventName: CreateAccountAssignment` / `DeleteAccountAssignment`) on a trail in the Identity Center Region. Use those for detection/audit, **not** as the revoke trigger (you would race yourself).

Drive the workflow from Step Functions + Scheduler. Optionally emit your own EventBridge event `elevate.granted` / `elevate.revoked` with `requestId` for downstream notifications.

---

## 4. Security caveats

**Bot credentials.** The Azure Bot app secret or Bot Framework token is equivalent to the ability to post spoofed “approved” cards if your backend trusts the card payload more than the **validated JWT**. Verify: tenant ID, audience = bot app ID, service URL host allow-list (`smba.trafficmanager.net` / Teams service URLs), and that `from.aadObjectId` maps to an approver **before** starting Grant. Store secrets in Secrets Manager; prefer certificate/federated credentials. Rotate.

**Who can invoke.** API Gateway must not be an open `sso:CreateAccountAssignment` button. AuthZ in this order: (1) valid Bot Framework/Entra token, (2) conversation is the expected team/channel id (allow-list), (3) actor in approver group, (4) actor ≠ requester, (5) permission set and accounts in eligibility policy, (6) duration ≤ max for that set/env. Channel membership is insufficient.

**Confused deputy.**

- Teams: an attacker in another tenant installing a lookalike app should fail tenant-id and conversation-id checks.
- AWS: the grant role’s trust policy should be Lambda/Step Functions in **this** account only (`aws:SourceAccount`, `aws:SourceArn`). Do not allow Chatbot, broad `lambda.amazonaws.com` without source ARN, or the management account to assume it.
- SNS/Workflows webhooks: if you also post notifications via a Workflows URL, treat that URL as a secret; it does not prove an approval.
- Do **not** attach `sso:CreateAccountAssignment` to an Amazon Q chat channel role. Even though `sso:*` is hard-denied in Chatbot CLI, a Lambda target of a custom action is a deputy if that Lambda is over-permissioned and invokable by the channel.

**Management account blast radius.** Never let the JIT role assign `AdministratorAccess` (or any permission set) on the management account. Delegated-admin TEAM **cannot** manage management-account assignments by design — keep that invariant. Permission sets themselves should be least-privilege (`first_responder` ≠ `OrganizationAccountAccessRole`). A bug in the bot is then bounded to the allow-listed sets and accounts.

**Standing vs temporary.** JIT `USER` assignments must not clobber IaC-managed `GROUP` assignments. Revoke only rows the workflow created (store the tuple in DynamoDB; never “delete all assignments for this user”).

**Session overhang.** Always assume access lasts `requested_duration + permission_set_session_duration + SCIM_lag` (the last term only for PIM). Document this for auditors.

**Data in Teams.** Justifications and account IDs in a standard channel are visible to everyone in that team. Do not put secrets in cards. Retention follows Teams / Microsoft 365 policy — keep the system of record in DynamoDB + CloudTrail.

**TEAM-specific (if used as engine).** AppSync without WAF by default; Cognito; Amplify S3 artifacts. Restrict TEAM admins. Do not deploy other workloads in the TEAM account. Read TEAM security docs before production.

---

## 5. TEAM + Microsoft Teams — exact facts

| Question | Answer |
| --- | --- |
| Native Teams support? | **No.** Notifications: SES, Slack (OAuth bot token in Settings), SNS topic `TeamNotifications-main`. |
| Native Slack support? | Yes, as of v1.1.0: install TEAM’s Slack app, paste Bot User OAuth Token. Token is a secret (can DM users, read emails). |
| Bot / Azure app registration? | None provided. |
| Request/approve UX | Web UI as Identity Center custom SAML 2.0 app (`TEAM IDC APP` in the access portal). |
| Grant mechanism | Step Functions Grant → Identity Center account assignment for the requester user. |
| Repo | https://github.com/aws-samples/iam-identity-center-team |
| Docs | https://aws-samples.github.io/iam-identity-center-team/ |
| Notifications config | https://aws-samples.github.io/iam-identity-center-team/docs/deployment/configuration/notifications.html |
| Architecture | https://aws-samples.github.io/iam-identity-center-team/docs/overview/architecture.html |
| Deploy | Delegated admin account; `init.sh` + `deploy.sh`; parameters include `IDC_LOGIN_URL`, `TEAM_ACCOUNT`, `TEAM_ADMIN_GROUP`, `TEAM_AUDITOR_GROUP`, `CLOUDTRAIL_AUDIT_LOGS`. |
| Teams integration today | Enable SNS → subscribe Lambda → post to Teams (Workflows or Graph). Approval remains in TEAM. |

---

## 6. Suggested rollout

1. Confirm Identity Center **org instance**, delegated admin account, Entra SAML + SCIM healthy, permission set `first_responder` (and peers) provisioned to target accounts with **1 hour** session duration.
2. Ship **break-glass** (direct SAML to emergency IAM roles) and test it **before** depending on the bot.
3. If a web console is enough for v1: deploy **TEAM** in delegated admin, turn on SNS, bridge to a Teams **notification** channel. Use this to learn eligibility/approval policy shape.
4. For the target UX: implement the **custom bot + Step Functions** path (section 2), reusing TEAM’s state model and Identity Center APIs. Keep Chatbot out of the grant role.
5. Add prod two-person rule, Scheduler revoke with DLQ alarms, CloudTrail Lake queries keyed by `requestId` / UserId, and a monthly break-glass exercise.
6. Adopt PIM-for-Groups only for **standing eligible groups** that are not incident-scoped (for example a permanent “security audit readonly” activation), not as a replacement for account-scoped war-room elevation.

---

## 7. Primary source URLs

### AWS TEAM and Identity Center JIT

- https://github.com/aws-samples/iam-identity-center-team
- https://aws-samples.github.io/iam-identity-center-team/
- https://aws-samples.github.io/iam-identity-center-team/docs/overview/architecture.html
- https://aws-samples.github.io/iam-identity-center-team/docs/deployment/deployment_process.html
- https://aws-samples.github.io/iam-identity-center-team/docs/deployment/configuration/notifications.html
- https://aws-samples.github.io/iam-identity-center-team/docs/overview/security.html (also `docs/docs/overview/security.md` in the repo)
- https://aws.amazon.com/blogs/security/temporary-elevated-access-management-with-iam-identity-center/
- https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/emergency-access.html
- https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/break-glass-access.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/delegated-admin.html

### Identity Center and Identity Store APIs

- https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_CreateAccountAssignment.html
- https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_DeleteAccountAssignment.html
- https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_DescribeAccountAssignmentCreationStatus.html
- https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_DescribeAccountAssignmentDeletionStatus.html
- https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_ListAccountAssignments.html
- https://docs.aws.amazon.com/singlesignon/latest/IdentityStoreAPIReference/API_GetUserId.html
- https://docs.aws.amazon.com/service-authorization/latest/reference/list_iam-identity-center.html
- https://aws.amazon.com/blogs/security/use-new-account-assignment-apis-for-aws-sso-to-automate-multi-account-access/
- https://docs.aws.amazon.com/singlesignon/latest/userguide/limits.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/logging-using-cloudtrail.html
- https://docs.aws.amazon.com/eventbridge/latest/ref/events-ref-sso.html
- https://aws.amazon.com/blogs/security/how-to-revoke-federated-users-active-aws-sessions/

### Amazon Q Developer in chat applications (Teams)

- https://docs.aws.amazon.com/chatbot/latest/adminguide/what-is.html
- https://docs.aws.amazon.com/chatbot/latest/adminguide/teams-setup.html
- https://docs.aws.amazon.com/chatbot/latest/adminguide/understanding-permissions.html (includes non-supported `sso:*` / `identitystore:*`)
- https://docs.aws.amazon.com/chatbot/latest/adminguide/chatbot-cli-commands.html
- https://docs.aws.amazon.com/chatbot/latest/adminguide/custom-actions.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-chatbot-microsoftteamschannelconfiguration.html
- https://aws.amazon.com/blogs/aws/aws-chatbot-now-integrates-with-microsoft-teams/

### Entra federation, SCIM, PIM

- https://docs.aws.amazon.com/singlesignon/latest/userguide/idp-microsoft-entra.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/provision-automatically.html
- https://learn.microsoft.com/en-us/entra/identity/saas-apps/aws-single-sign-on-provisioning-tutorial (includes “JIT application access with PIM for groups”)
- https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/concept-pim-for-groups
- https://aws.amazon.com/blogs/security/implementing-just-in-time-privileged-access-to-aws-with-microsoft-entra-and-aws-iam-identity-center/

### Teams bots and Adaptive Cards

- https://learn.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/universal-actions-for-adaptive-cards/work-with-universal-actions-for-adaptive-cards
- https://learn.microsoft.com/en-us/microsoftteams/platform/task-modules-and-cards/cards/universal-actions-for-adaptive-cards/user-specific-views
- https://learn.microsoft.com/en-us/adaptive-cards/authoring-cards/universal-action-model
- https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent
- https://learn.microsoft.com/en-us/azure/bot-service/bot-service-overview
- https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook (Workflows replacement for retired connectors)

### Scheduler, Organizations, partners

- https://docs.aws.amazon.com/scheduler/latest/APIReference/API_CreateSchedule.html
- https://aws.amazon.com/blogs/compute/automatically-delete-schedules-upon-completion-with-amazon-eventbridge-scheduler/
- https://docs.aws.amazon.com/cli/latest/reference/organizations/list-accounts-for-parent.html
- https://aws.amazon.com/about-aws/whats-new/2023/05/aws-partners-temporary-elevated-access-capabilities-iam-identity-center/
- https://goteleport.com/docs/identity-governance/access-requests/plugins/msteams/
- https://goteleport.com/docs/identity-governance/integrations/aws-iam-identity-center/

---

## 8. One-page decision

- Need **approve in Teams** for account-scoped Identity Center permission sets → **custom bot + Step Functions + `CreateAccountAssignment` / `DeleteAccountAssignment`**, Scheduler for expiry. That is the recommended path.
- Need **AWS-supported JIT this quarter** and can live with a web UI → **TEAM**, SNS into Teams for awareness only.
- Already **Entra PIM-centric** and elevation is group-shaped → **PIM for Groups + SCIM**, accept lag and lingering sessions; not a Teams approval workflow.
- Do **not** use Amazon Q Developer in chat applications to elevate Identity Center privileges.
- Keep **break-glass** off Teams, off the bot, and preferably off Identity Center’s single Region.
