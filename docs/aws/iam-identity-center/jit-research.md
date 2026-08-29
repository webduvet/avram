# AWS just-in-time (JIT) / temporary elevated privilege access

**Research brief for a multi-account AWS Organization (management / “tower” account + member accounts)**

- **As-of date:** 29 August 2026 (Europe/Dublin)
- **Scope:** What AWS recommends, what AWS provides, and what still has to be built or operated.
- **Sources:** Primary AWS documentation, AWS official blogs/samples, and the AWS-samples TEAM repository. Third-party recaps are not used as authority.
- **Style:** Impersonal. No company names or people.

---

## 1. Executive summary

AWS’s recommended workforce access model for a multi-account Organization is:

1. **IAM Identity Center** (organization instance) as the single federation point.
2. **Permission sets** as the unit of AWS account entitlement.
3. **Standing access that is least-privilege** (typically read-only or scoped operator sets).
4. **Temporary elevated access (JIT)** for privileged permission sets such as `AdministratorAccess`, `PowerUserAccess`, or a custom incident-responder set.
5. **Emergency / break-glass access** as a **separate last-resort path**, independent of Identity Center, used only when Identity Center, the IdP, or the Identity Center Region is unavailable.

**Critical finding (GA status, 2025–2026):** IAM Identity Center does **not** have a native, fully managed temporary-elevated-access *service feature* as of 29 August 2026. Official Identity Center documentation still frames JIT as something Identity Center *integrates with*:

- **Vendor-managed** solutions from AWS Security Competency partners, **or**
- **Self-managed** solutions, of which the AWS-samples **TEAM** application is the official open-source reference.

The grant/revoke *mechanism* Identity Center itself provides is programmatic **account assignment** (`CreateAccountAssignment` / `DeleteAccountAssignment`) plus permission-set **session duration**. Request, approval, eligibility, time-boxing of the *assignment*, and auto-revoke are **not** native Identity Center product features. They are implemented by TEAM, by a partner PAM product, or by custom orchestration.

A previous operating pattern — request elevated privileges via chat, human approval, automatic time-boxed role in target account(s), accountability via reason + approval + time box + audit — is aligned with AWS’s published model. AWS does not ship that chat-request/approval loop as a managed Identity Center feature. Chat request/approval is a **separate** sample path (Systems Manager Change Manager + Amazon Q Developer in chat applications, formerly AWS Chatbot). TEAM’s own docs and the ChatOps sample both state that TEAM **notifies** via chat/SNS; it does **not** (as of the sample’s own wording) raise or approve requests *in* Slack/Teams.

---

## 2. Native Identity Center JIT vs TEAM vs partners — GA status

This is the decision that everything else hangs on. Dates below are from AWS “What’s New”, Identity Center docs, and the TEAM repo as fetched on 29 August 2026.

| Capability | Status as of 29 Aug 2026 | What it actually is |
|---|---|---|
| IAM Identity Center **native temporary elevated access** (managed product feature: request/approve/time-box/revoke in the Identity Center console/API) | **Not GA. Does not exist as a first-party Identity Center feature.** | Official page [Temporary elevated access for AWS accounts](https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html) describes JIT as a *pattern*, then points to **partner solutions** and (in SRA / TEAM docs) **self-managed** solutions. There is no Identity Center console workflow, no `StartTemporaryAccess` API, and no What’s New item announcing a native JIT feature in 2025 or 2026. |
| Partner JIT integrations | **GA since 22 May 2023** | [What’s New: AWS partners bring choice of temporary elevated access capabilities to IAM Identity Center](https://aws.amazon.com/about-aws/whats-new/2023/05/aws-partners-temporary-elevated-access-capabilities-iam-identity-center/). Current Identity Center doc lists validated solutions: Apono, CyberArk Secure Cloud Access, Okta Access Requests, Tenable (previously Ermetic). |
| TEAM (aws-samples) | **Open-source sample / solution, not a managed AWS service.** Latest release **v1.5.0 (26 June 2026)**. | [github.com/aws-samples/iam-identity-center-team](https://github.com/aws-samples/iam-identity-center-team). Deployed, operated, patched, and secured by the customer. Explicit AWS Content / sample disclaimer. |
| IAM **temporary delegation** (Nov 2025) | **GA 19 Nov 2025 — different problem** | [What’s New](https://aws.amazon.com/about-aws/whats-new/2025/11/streamline-integration-partner-products-iam-delegation/). Lets customers grant **limited, temporary access to Amazon/AWS Partner *products*** for onboarding/maintenance. **Not** workforce JIT into member accounts. |
| IAM **account access manager** | **GA 10 Aug 2026 — different problem** | [What’s New](https://aws.amazon.com/about-aws/whats-new/2026/08/aws-iam-aam/). Assigns *existing IAM roles* in accounts to Identity Center users/groups. Complements permission sets; does not add request/approval/time-box. |
| Identity Center multi-Region replication | **GA 3 Feb 2026** | Improves resilience of *standing* entitlements if the primary Identity Center Region is disrupted. Does **not** replace break-glass. |

**Authoritative wording (Identity Center User Guide):**

> Temporary elevated access (also known as just-in-time access) is a way to request, approve, and track the use of a permission to perform a specific task during a specified time. Temporary elevated access supplements other forms of access control, such as permission sets and multi-factor authentication. […] To address a range of customers’ needs, AWS IAM Identity Center integrates with the solutions from AWS Security Competency partners. […] Validated solutions include Apono Access Management Platform, CyberArk Secure Cloud Access, Okta Access Requests, and Tenable (previously Ermetic).

Source: [docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html](https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html)

**Authoritative wording (AWS Security Reference Architecture — Identity Management):**

> IAM Identity Center supports integration with temporary elevated access management (TEAM) solutions (also known as just-in-time access). […] IAM Identity Center supports both vendor-managed TEAM solutions from supported AWS security partners or self-managed solutions, which you maintain and tailor to address your time-bound access requirements.

Source: [workforce-iam-identity-center.html](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture-identity-management/workforce-iam-identity-center.html)

**Implication:** A team that wants AWS-aligned JIT in 2026 chooses among:

1. **Operate TEAM** (or a fork) as a self-managed application.
2. **Buy a validated partner PAM** that drives Identity Center assignments.
3. **Build custom orchestration** on `sso-admin` APIs (Lambda/Step Functions/ITSM/chat).
4. **Keep standing elevated assignments** (not recommended by AWS for production privilege).

There is no “turn on Identity Center JIT in the console” option.

---

## 3. IAM Identity Center primitives (what AWS *does* provide)

### 3.1 Permission sets

A permission set is a **template of IAM policies**, created once in the Identity Center organization instance and **provisioned into each assigned account as an Identity Center-controlled IAM role**. Identity Center manages the role; users assume it via the AWS access portal or AWS CLI.

Sources:

- [Manage AWS accounts with permission sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html)
- [Assign user or group access to AWS accounts](https://docs.aws.amazon.com/singlesignon/latest/userguide/assignusers.html)

A permission set can include:

- AWS managed policies (including job-function policies such as `AdministratorAccess`, `PowerUserAccess`, `ReadOnlyAccess`, `ViewOnlyAccess`)
- Customer managed policies
- One inline policy (max 32,768 bytes; 10,240 non-whitespace)
- An AWS managed or customer managed policy as a **permissions boundary**

AWS’s own least-privilege guidance for permission sets:

- After creating an administrative set, create a **more restrictive** set and assign it as well.
- Administrative users should also have a restrictive set so they can choose it instead of always using admin.
- Use IAM Access Analyzer to observe actual API usage, then replace AWS managed job-function policies with custom policies.
- When signing into the access portal, **choose the most restrictive role that still does the job**.

### 3.2 Assignment model (accounts vs OUs)

Identity Center **account assignments are per AWS account**, not per OU.

- Console: select accounts (up to **10 accounts at a time** per permission set), then users/groups, then permission sets.
- API: `CreateAccountAssignment` / `DeleteAccountAssignment` with `TargetType = AWS_ACCOUNT` and `TargetId = 12-digit account ID`.
- There is **no native “assign this permission set to everyone in this OU, permanently or temporarily”** Identity Center API. OU targeting is a **TEAM eligibility-policy convenience** (TEAM expands an OU to member accounts, and as of v1.5.0 can cache that mapping).
- TEAM eligibility that specifies an OU includes **only accounts directly in that OU, not child OUs**.

**Management-account extra restriction:** assigning access to the Organizations **management account** requires `IAMFullAccess` or equivalent in that account. That extra restriction does **not** apply to member accounts.

**Group vs user assignment for the management account (AWS Identity Center best practice):** assign **users, not groups**, to management-account permission sets. Anyone who can mutate group membership (IdP admin, AD admin, Identity Center admin) would otherwise be able to grant management-account access without an Identity Center assignment change. If groups are used, IdP-side controls and logging of membership change are mandatory.

Sources:

- [Assign user or group access](https://docs.aws.amazon.com/singlesignon/latest/userguide/assignusers.html)
- [CreateAccountAssignment API](https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_CreateAccountAssignment.html)
- [Delegated administration — best practices](https://docs.aws.amazon.com/singlesignon/latest/userguide/delegated-admin.html)

### 3.3 Session duration (two clocks)

There are **two independent clocks**. Confusing them is a common operational failure of JIT.

| Clock | What it controls | Default | Range | Source |
|---|---|---|---|---|
| **Permission-set / IAM role session** | How long console/CLI/SDK credentials for that permission set remain valid **after the user federates into the account** | 1 hour | **1–12 hours** | [Set session duration for AWS accounts](https://docs.aws.amazon.com/singlesignon/latest/userguide/howtosessionduration.html) |
| **AWS access portal session** | How long the user stays signed into the Identity Center portal (and can launch new account sessions / CLI) without re-authenticating | 8 hours | 15 minutes – 90 days | [Configure the session duration in IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/configure-user-session.html) (linked from permission-set docs); Security Blog [Define a custom session duration and terminate active sessions](https://aws.amazon.com/blogs/security/define-a-custom-session-duration-and-terminate-active-sessions-in-iam-identity-center/) |

AWS security best practice on permission-set duration: **do not set it longer than needed to perform the role.**

**JIT-specific implication (documented by TEAM):** TEAM’s requested duration is the **assignment window** (when the permission set is attached to the user for that account). It does **not** cut off already-issued IAM role sessions. A session started one minute before revoke remains valid until the permission-set session duration expires. TEAM therefore recommends keeping elevated permission-set session duration at the **default 1 hour**. Terminating the portal session does **not** terminate an in-flight permission-set session.

IAM Identity Center automatically creates the backing IAM roles with a **maximum session duration of 12 hours**; the permission-set setting is what is actually issued.

### 3.4 Recommended permission-set model (standing vs elevated)

Aligned with Identity Center docs, Well-Architected SEC03-BP02, and TEAM eligibility design:

| Class | Examples | Standing assignment? | Session duration | Who gets it |
|---|---|---|---|---|
| **Standing — read / observe** | `ReadOnlyAccess`, `ViewOnlyAccess`, custom “prod-read”, Security Audit | Yes, to groups, scoped to the accounts those groups actually operate | 1–4 hours | Developers, SREs, auditors — day-to-day |
| **Standing — scoped build (non-prod)** | Custom “dev-power”, no IAM/Organizations/IdC admin | Yes, **non-production OUs only** | 4 hours typical | Developers in sandbox/dev |
| **Elevated — change production** | Custom “incident-responder”, `PowerUserAccess` (no IAM), narrowly scoped ops sets (e.g. ECS/EKS deploy, RDS restore) | **No standing assignment.** Eligible to *request* via TEAM/partner/custom JIT | **1 hour** (TEAM recommendation) | Eligible on-call / platform / app owners, after approval |
| **Elevated — admin** | `AdministratorAccess` or a near-equivalent | **No standing assignment in member accounts. Never standing in the management account for a broad group.** | **1 hour** | Small eligible set; approval required; short max TEAM duration |
| **Management-account org admin** | Dedicated, separate permission sets used **only** in the management account | Standing only for the smallest possible named user set; prefer JIT even here (Identity Center delegated-admin docs explicitly say to consider temporary elevated access for management-account access) | 1 hour | Named humans, not groups |
| **Break-glass** | Direct IAM federation / IAM users in an emergency account — **not** Identity Center permission sets | Pre-created, unused, alarmed | Independent of Identity Center | See §7 |

Do **not** model a single `first_responder` permission set that covers **all production accounts** as a standing assignment. That is standing admin with a friendly name. The AWS-aligned equivalent is:

- One (or a few) **elevated permission sets**, provisioned (the *role exists* in each prod account) but **not assigned** to people until a request is approved.
- TEAM **eligibility** may list many accounts or an OU so a responder can *request* them.
- Each request still names **an account + a permission set + a duration + a justification**. TEAM does not grant “all prod accounts at once” as a single assignment; it grants the requested account. Multi-account incidents require multiple requests, or a custom wrapper.

Provisioning a permission set into an account (so the IAM role exists) is not the same as assigning a principal to it. JIT operates on **assignments**.

---

## 4. TEAM — architecture, workflow, placement

### 4.1 What TEAM is

**Temporary elevated access management (TEAM) for AWS IAM Identity Center** is an open-source, Amplify-hosted React SPA plus serverless backend that:

- Lets eligible users **request** a permission set on a specific account for a time window, with justification.
- Routes the request to **approver groups**.
- On approval, at start time, calls Identity Center to **create** the account assignment.
- When the window ends (or on revoke), **deletes** the assignment.
- Surfaces request history and, via **CloudTrail Lake**, session activity during the window.

It is **AWS Content / sample code**. The customer is responsible for testing, securing, operating, and patching it for production use.

Primary sources:

- GitHub: [https://github.com/aws-samples/iam-identity-center-team](https://github.com/aws-samples/iam-identity-center-team)
- Docs site: [https://aws-samples.github.io/iam-identity-center-team/](https://aws-samples.github.io/iam-identity-center-team/)
- Architecture: [https://aws-samples.github.io/iam-identity-center-team/docs/overview/architecture.html](https://aws-samples.github.io/iam-identity-center-team/docs/overview/architecture.html)
- Workflow: [https://aws-samples.github.io/iam-identity-center-team/docs/overview/workflow.html](https://aws-samples.github.io/iam-identity-center-team/docs/overview/workflow.html)
- Security: [https://aws-samples.github.io/iam-identity-center-team/docs/overview/security.html](https://aws-samples.github.io/iam-identity-center-team/docs/overview/security.html)
- Policies: [https://aws-samples.github.io/iam-identity-center-team/docs/overview/policies.html](https://aws-samples.github.io/iam-identity-center-team/docs/overview/policies.html)
- Prerequisites: [https://aws-samples.github.io/iam-identity-center-team/docs/deployment/prerequisites.html](https://aws-samples.github.io/iam-identity-center-team/docs/deployment/prerequisites.html)
- Deployment: [https://aws-samples.github.io/iam-identity-center-team/docs/deployment/deployment_process.html](https://aws-samples.github.io/iam-identity-center-team/docs/deployment/deployment_process.html)
- Security Blog (introduction): [https://aws.amazon.com/blogs/security/temporary-elevated-access-management-with-iam-identity-center/](https://aws.amazon.com/blogs/security/temporary-elevated-access-management-with-iam-identity-center/)
- Conceptual precursor (identity-broker pattern, 2021): [https://aws.amazon.com/blogs/security/managing-temporary-elevated-access-to-your-aws-environment/](https://aws.amazon.com/blogs/security/managing-temporary-elevated-access-to-your-aws-environment/)
- Latest release: [v1.5.0 (26 June 2026)](https://github.com/aws-samples/iam-identity-center-team/releases/tag/1.5.0)

### 4.2 Personas

Determined by IAM Identity Center **group membership**, synchronized into Amazon Cognito for app authorization:

| Persona | Who | What they can do |
|---|---|---|
| **Requester** | Users (or groups) with an **eligibility policy** | Request eligible account + permission set; see own history |
| **Approver** | Members of groups named in an **approver policy** for that account/OU | Approve/reject; cannot approve **their own** request; can also be requesters |
| **Auditor** | Dedicated TEAM auditor Identity Center group | Global view of requests, justifications, and session logs |
| **Admin** | Dedicated TEAM admin Identity Center group | App settings (max duration, mandatory fields, approval on/off), eligibility policies, approver policies |

### 4.3 Request lifecycle and how access is granted/revoked

Request states: **Pending → Approved | Rejected | Expired | Cancelled**; then **Scheduled → In progress → Ended | Revoked**; or **Error**.

Orchestration (DynamoDB stream → Lambda “TEAM router” → Step Functions):

1. **Approval workflow** (new Pending request): notify approver group; wait a configurable period (default **1 hour**); if untouched, mark **Expired**.
2. **Reject workflow**: notify requester (and approvers if cancelled).
3. **Schedule workflow** (on approve): notify requester; wait until requested start time; invoke Grant.
4. **Grant workflow**: **`CreateAccountAssignment`** — Identity Center user + permission set + target account. Status **in progress**. Notify requester. Wait requested duration. Invoke Revoke.
5. **Revoke workflow** (timer or manual revoke by requester/approver): **`DeleteAccountAssignment`**. Notify requester. Status **ended**.

During the in-progress window the requester uses the **normal Identity Center access portal / AWS CLI** to start sessions with that permission set. TEAM does not mint STS credentials itself.

### 4.4 Components (what is deployed where)

TEAM is a **full-stack serverless SPA in one account** (the TEAM / Identity Center delegated-admin account). It does **not** deploy a stack into every member account. Member-account effects are entirely via Identity Center provisioning of permission-set IAM roles (which Identity Center already does).

| Layer | AWS services |
|---|---|
| UI | React SPA, hosted by **AWS Amplify**; accessed as a **custom SAML 2.0 application** on the Identity Center access portal |
| AuthZ | **Amazon Cognito** + SAML federation from Identity Center; Cognito groups mapped from Identity Center groups |
| API | **AWS AppSync** (GraphQL) |
| State | **DynamoDB**: Requests, Approvers, Eligibility, Session (CloudTrail Lake query state), Settings, OU Accounts Cache (v1.5.0) |
| Orchestration | **Lambda** router + **Step Functions** (Approval, Reject, Schedule, Grant, Revoke) |
| Org/IdC reads | Lambda resolvers: `teamgetAccounts`, `teamgetOUs`, `teamgetPermission`, `teamgetUsers`, `teamgetIdcGroups`; v1.5.0 adds `teamgetOUAccounts`, `teaminvalidateOUCache`, `teamvalidateRequest` |
| Audit in-app | **CloudTrail Lake** organization event data store (queried for the elevation window + 1 extra hour) |
| Notifications | Step Functions notify approvers/requesters (SNS-backed; chat is notification-only unless customised) |

**v1.5.0 (26 June 2026)** adds an OU→account cache because Organizations APIs default to **5 TPS**, which made the request form slow in large orgs with OU-level eligibility. Cache is **off by default**; TTL default 1 week; stale cache must be invalidated in the UI when accounts move.

### 4.5 Management account vs TEAM (delegated admin) account vs member accounts

| Account | What is deployed / allowed |
|---|---|
| **Organizations management (“tower”) account** | Identity Center **instance always lives here**. TEAM **should not** be deployed here. `init.sh` registers the TEAM account as delegated administrator for **IAM Identity Center**, **CloudTrail Lake**, and **Account Management**. TEAM **cannot** (by Identity Center delegated-admin design) enable/disable user access **in the management account** or manage permission sets **provisioned in the management account**. |
| **Dedicated TEAM / IdC delegated-admin account** | Entire TEAM application. No other workloads. Least-privilege human access. This is also the CloudTrail Lake delegated admin. |
| **Member / workload accounts** | No TEAM resources. Identity Center-controlled IAM roles for each **provisioned** permission set. JIT only changes **who is assigned** to those roles, and for how long. |

Deploying TEAM into the management account is documented as a discouraged exception (`parameters-mgmt-template.sh`, skip `init.sh`). AWS’s wording: proceed with caution.

### 4.6 Eligibility and approval policies

**Eligibility policy** (who may request):

- Entity: Identity Center **user or group**
- Scope: one or more **accounts and/or OUs** (OU = direct child accounts only)
- Permissions: one or more **permission sets**
- **Maximum duration:** 1–**8000 hours** (~1 year)
- **Approval required:** per policy; can also be turned off globally in TEAM settings

**Approver policy** (who may approve):

- Account or OU
- One or more Identity Center groups

If eligibility requires approval for an account, an approver policy for that account must exist and its groups must have members; otherwise TEAM will not allow the request (unless global “approval required” is off).

Peer approval (same group for eligibility and approval) is explicitly supported; **self-approval is not**.

### 4.7 Security history that operators must know

**CVE-2025-1969** (AWS Security Bulletin [AWS-2025-004](https://aws.amazon.com/security/security-bulletins/AWS-2025-004/), published 4 March 2025):

- Improper input validation allowed a user to modify a valid request and **spoof an approval**.
- Affected TEAM versions **< 1.2.2**.
- GitHub advisory: [GHSA-x9xv-r58p-qh86](https://github.com/aws-samples/iam-identity-center-team/security/advisories/GHSA-x9xv-r58p-qh86).
- Fix: upgrade to **1.2.2 or later** (current latest is 1.5.0). Follow TEAM “Update TEAM solution” docs.
- This is the operational reality of running a sample as production PAM: **patching is on the customer**.

Other TEAM security notes from TEAM’s own security page:

- Treat the TEAM account as highly privileged; no other workloads.
- AppSync WAF is **not** enabled by default; AWS WAF is recommended.
- Amplify artifact S3 bucket: enable server access logging (org-specific destination, often the log-archive account).
- Regional dependency: Cognito + Identity Center are regional; TEAM is unavailable if that Region’s Identity Center is disrupted → **break-glass is mandatory** (see §7).
- TEAM is **not** a substitute for IAM governance: it will happily time-box an over-permissive permission set.

### 4.8 What TEAM does **not** do

- It is **not** a managed AWS service (no AWS SLA, no AWS-operated patching, MIT-0 sample disclaimer).
- It **cannot** JIT into the **management account** when deployed as delegated admin (Identity Center platform limitation, not a TEAM bug).
- It does **not** recursively include nested OUs.
- It does **not** revoke in-flight IAM role **sessions** when the assignment is deleted.
- It does **not** provide chat **request/approve** UX (notifications only, per the related AWS-samples JIT/Chatbot repo).
- It does **not** enforce least privilege inside the permission set.
- It does **not** replace break-glass.
- It does **not** log data events by default (CloudTrail Lake EDS: management events only).
- In-app session log search depends on **CloudTrail Lake**. **CloudTrail Lake is closed to new customers as of 31 May 2026**; existing Lake customers can continue. New Organizations deploying TEAM after that date cannot create a new organization event data store. Session-log UI would need to be re-pointed at CloudTrail **trails + Athena** (or Lake if the org already had it). See [Working with AWS CloudTrail Lake](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake.html).
- Default Amplify deployment includes a non-trivial surface (Cognito, AppSync, many Lambdas). Production use requires aligning it with the org’s SDLC, CIS, and WAF standards.

---

## 5. AWS Organizations / Control Tower implications

### 5.1 Why standing admin in the management account is the wrong control

Official Organizations guidance ([Best practices for the management account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices_mgmt-acct.html)):

- Limit who has access to the management account.
- Use it **only** for tasks that **require** the management account.
- **Avoid deploying workloads** there.
- **SCPs do not apply** to principals in the management account. Standing admin there is unconstrained by the guardrails that protect every other account.
- Delegate administration of services (Identity Center, CloudTrail, GuardDuty, Config, Security Hub, etc.) to member accounts.

Identity Center delegated-admin docs repeat the same point and add: **consider temporary elevated access even for the remaining management-account tasks**, use **dedicated permission sets** for that account, and assign **users not groups**.

Control Tower: the Control Tower **console** is only for management-account administrators. Member-account users should not need it. Control Tower preventive controls are SCPs/RCPs/declarative policies — they also **do not constrain the management account**.

Landing-zone implication: a Control Tower “tower” / management account is for Organizations, Control Tower, Billing, and (historically) enabling Identity Center. **Identity Center administration should be delegated.** TEAM belongs in that delegated-admin (or a dedicated identity) account in a Security OU, not in the management account, and not in a workload account.

### 5.2 SCPs and JIT

SCPs set the **maximum** permissions for IAM users and roles in **member** accounts; they **grant nothing**. Effective permission = identity policy ∩ permissions boundary ∩ session policy ∩ SCP ∩ RCP.

Implications for JIT:

- An SCP that `Deny`s `iam:*` / `organizations:*` / `account:*` in prod OUs **still applies** to Identity Center-provisioned roles, including `AdministratorAccess` permission sets. That is desirable: JIT admin cannot disable CloudTrail, leave the org, or create unmanaged IAM users if the SCP forbids it.
- SCPs **cannot** be used as the JIT switch itself (they don’t attach/detach permission sets, and they don’t apply to the management account).
- SCPs **do** apply to **delegated-admin** accounts (including the TEAM account). Guard the TEAM account with SCPs that prevent it from becoming a general-purpose admin account, while still allowing the Identity Center and CloudTrail APIs TEAM needs.
- You **cannot** SCP-deny standing admin in the management account. Reduce standing admin there by **not assigning** broad permission sets, using JIT/partners for remaining tasks, and break-glass for IdC failure.
- Service-linked roles are **not** restricted by SCPs.
- Recommended complementary SCP (Organizations docs): deny `organizations:LeaveOrganization` and `account:CloseAccount` at the root. Organizations created in the console after **10 July 2026** get this by default; older orgs must attach it.

A realistic “block standing admin, allow JIT” design is **not** an SCP that magically allows TEAM-granted roles and denies others. It is:

1. **Do not assign** elevated permission sets standing.
2. SCPs **cap** what even a JIT `AdministratorAccess` session can do (protect audit, IdC, org, root, CloudTrail, Config).
3. TEAM/partner is the only process that creates the assignment.

### 5.3 Permission boundaries

Permission sets can attach a **permissions boundary**. The boundary is the max the Identity Center role can ever exercise, even if the permission set’s identity policy is `AdministratorAccess`. Useful for elevated sets that should never manage IAM or KMS CMKs, for example.

Boundaries do not grant permissions. They also do not apply to the management account’s lack of SCPs. They are an in-account ceiling, complementary to SCPs.

Source: [Permissions boundaries for IAM entities](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html)

---

## 6. Related AWS primitives (how they fit JIT)

### 6.1 IAM role session duration and STS

Workforce JIT in this architecture should **not** be implemented as humans calling `sts:GetSessionToken` or long-lived IAM users. IAM best practices: **humans federate; workloads use roles**.

| STS API | Role in this design |
|---|---|
| Identity Center federation (OIDC/SAML under the hood, then role assumption into the permission-set role) | **Primary.** Session duration = permission-set setting (1–12 h). |
| `AssumeRole` | Used by Identity Center-generated roles; also the break-glass cross-account pattern. Role chaining caps duration at **1 hour**. Max role session 1–12 h (IAM), 15 min–12 h depending on method. [Methods to assume a role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use.html) |
| `GetSessionToken` | MFA-protected **IAM user** sessions (15 min–36 h; root max 1 h). Relevant only to **break-glass IAM users**, not Identity Center JIT. [Request temporary security credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_request.html) |
| `GetFederationToken` | Custom identity brokers. TEAM does **not** use this; it uses Identity Center assignments. |
| `AssumeRoleWithSAML` | Direct IdP→IAM federation for **emergency access** when Identity Center is down. |

Temporary credentials expire; they are not stored on the user. That is the whole point of both standing Identity Center access and JIT.

### 6.2 CloudTrail — “who had what when”

Three layers are needed; TEAM only automates the third for Lake customers.

1. **Identity Center / SSO APIs in the management (or delegated-admin) account:** `CreateAccountAssignment`, `DeleteAccountAssignment`, permission-set changes. These are **management events**. They answer *when was this human entitled to this permission set on this account*.
2. **Management events in the target account:** `AssumeRole` / console login into the Identity Center-provisioned role, then `ec2:TerminateInstances`, `iam:CreateUser`, etc. These answer *what they did*.
3. **Data events (optional, costed):** S3 object APIs, Lambda `Invoke`, DynamoDB item APIs, etc. TEAM’s Lake datastore **does not include data events by default**. Enable them where compliance requires object-level accountability (many AWS Config conformance packs include `cloudtrail-s3-dataevents-enabled`). [Logging data events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html)

**CloudTrail Lake:** TEAM queries an **organization event data store** in the TEAM account for the elevation window **plus one extra hour** (to cover leftover sessions). Lake is a SQL query engine over ORC-converted events; retention up to ~7 or ~10 years depending on pricing option. **New-customer signup closed 31 May 2026.** Alternative for new deployments: organization **CloudTrail trail** to the log-archive account + **Athena** (or CloudTrail Lake if the org already has an EDS).

Control Tower / SRA already send organization trails to a **Log Archive** account. Prefer querying that, rather than creating a second ingest path, unless TEAM’s in-app viewer is a hard requirement and Lake already exists.

### 6.3 AWS Config

Config does not grant or revoke access. It is the **detective** control that standing-privilege and JIT both need:

- Record `AWS::SSO::*` / account assignment change is **not** the primary IdC assignment record — CloudTrail is.
- Record IAM roles, role policies, and Identity Center-provisioned role updates in member accounts.
- Organizational Config rules / Control Tower detective controls for: IAM user presence (should be near-zero in member accounts), MFA, CloudTrail enabled, public access, etc.
- Conformance packs for Well-Architected Security often require S3 data events.

[Evaluating resources with AWS Config rules](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config.html)

### 6.4 IAM Access Analyzer

- **External / internal access analyzers:** detect resource policies that punch through account boundaries — relevant because a JIT admin could attach a bad resource policy.
- **Unused access analyzer:** find standing permission sets/roles that are never used — evidence to remove standing admin.
- **Policy generation:** build least-privilege customer-managed policies from CloudTrail, then attach those to elevated permission sets instead of `AdministratorAccess`.
- Delegated administrator for Access Analyzer is recommended (do not run this from the management account as a day-to-day tool).

[Getting started with IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-getting-started.html)

### 6.5 ABAC

Identity Center supports passing attributes as session tags (`aws:PrincipalTag`) for fine-grained permission-set policies. Useful to *scope* an elevated set (e.g. only resources tagged `Team=foo`) so JIT is not “admin of the world.” ABAC should only be used when both principals and resources are in the Organization (external parties can reuse tag keys).

[Enable and configure attributes for access control](https://docs.aws.amazon.com/singlesignon/latest/userguide/configure-abac.html)

---

## 7. Break-glass vs JIT (they are not the same)

AWS documents these as **two different processes**.

| | **JIT / temporary elevated access** | **Emergency / break-glass access** |
|---|---|---|
| **When** | Normal operations: incident, change, debug. Identity Center and IdP are **up**. | Identity Center down, IdP down, federation metadata expired, Region event, TEAM/Cognito down, misconfiguration that locks operators out. |
| **Path** | Identity Center assignment, time-boxed, approved. | **Independent** of Identity Center: direct SAML to IAM in an emergency account, or pre-created MFA IAM users in that account, then `AssumeRole` into pre-created emergency roles in workload accounts. |
| **AWS docs** | [Temporary elevated access](https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html); TEAM | [Set up emergency access to the AWS Management Console](https://docs.aws.amazon.com/singlesignon/latest/userguide/emergency-access.html); [SEC03-BP03 Establish emergency access process](https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_permissions_emergency_process.html) |
| **Standing?** | Eligibility standing; **permission standing no** | Roles/users **pre-created** (control-plane independence) but **unused** and alarmed |
| **Use for daily elevation?** | Yes | **Anti-pattern.** Well-Architected lists “emergency processes used in non-emergency situations” as a common anti-pattern. |

Well-Architected SEC03-BP03 failure modes:

1. **IdP unavailable** — emergency account IAM users (few, named, MFA) or vaulted root of the *emergency* account (worse, shared).
2. **IdP configuration on AWS modified/expired** (bad SAML cert) — identity admins use emergency path into the Identity Center admin account to repair federation.
3. **Identity Center or Region disruption** — **direct SAML from the IdP to IAM** in the emergency account (works only if IdP *and* IAM data plane are up). If the identity source is Identity Center directory or AD, follow standard break-glass IAM-user guidance instead, because that directory is also Regional.

Implementation rules from SEC03-BP03:

- Pre-create the emergency account, IAM IdP, roles, and **cross-account emergency roles in every workload account** (StackSets). Do not depend on control-plane create APIs during the incident.
- SCP-protect those cross-account roles from deletion/modification.
- Empty emergency IdP groups in normal times; add members only during a declared emergency (or keep IAM users offline in a vault).
- CloudTrail + EventBridge alerts on **any** console login or `AssumeRole` in the emergency account **outside** a tracked incident.
- Dual control / vault; rotate credentials after use.
- Test in incident-response game days (SEC10-BP07).
- Root of the **management** account remains the last last-resort (Organizations, Control Tower, Identity Center enablement cannot be fully delegated). Keep it vaulted, MFA, never used for JIT.

TEAM’s own security page: if regional service events are a concern, set up emergency access to the console. TEAM is **not** break-glass.

AWS also now has **centralized root access / privileged root sessions** for member accounts (`sts:AssumeRoot`, task-scoped, max 15 minutes) from the management or a delegated admin. That is a **third** path for a handful of root-only member-account tasks (e.g. locked S3 bucket policy), not a substitute for workforce JIT. Source: [Secure root user access for member accounts](https://aws.amazon.com/blogs/security/secure-root-user-access-for-member-accounts-in-aws-organizations/).

---

## 8. Well-Architected Security Pillar and IAM best practices

Mapped to this design:

| Practice | Source | Application to JIT |
|---|---|---|
| Humans use federation and **temporary credentials**; no long-lived IAM users for workforce | [IAM security best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) | Identity Center for all human AWS access |
| Workloads use **roles**, not user keys | same | Out of scope for this brief, but don’t mix JIT humans with embedded keys |
| MFA | same | Enforced at IdP or Identity Center; break-glass users hardware MFA |
| **Least privilege**; start from managed job-function policies, then tighten with Access Analyzer | same; [SEC03-BP02](https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_permissions_least_privileges.html) | Standing read-only; elevated sets as narrow as possible; `AdministratorAccess` only as a transitional elevated set |
| **Users have limited, time-bound production access; revoke immediately after the task** | SEC03-BP02 desired outcome, almost verbatim | This *is* JIT |
| Restrict administrator privileges to a small trusted group | SEC03-BP02 | Eligibility group ≠ standing assignment. “Small admin group” is necessary but **not sufficient** (see §13) |
| Permissions boundaries to delegate IAM | IAM BP | Optional ceiling on elevated permission sets |
| SCPs / RCPs as org-wide guardrails | IAM BP; Organizations | Cap blast radius of JIT admin |
| Emergency access process, tested, logged, not used daily | [SEC03-BP03](https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_permissions_emergency_process.html) | Separate from TEAM |
| Don’t use root for daily work | IAM BP | Root ≠ JIT ≠ break-glass IAM user |

SEC03-BP02 desired outcome (quote-level paraphrase from the live doc): separate AWS accounts isolate developers from production; when developers need production, they receive **limited, controlled access only for the duration of those tasks**, then access is **immediately revoked**.

That is the Well-Architected justification for replacing a standing `first_responder` assignment with JIT.

---

## 9. Amazon Q Developer in chat applications (formerly AWS Chatbot)

**Not part of TEAM.** Separate AWS service and a separate AWS-samples path.

- Service docs: [What is Amazon Q Developer in chat applications?](https://docs.aws.amazon.com/chatbot/latest/adminguide/what-is.html) — AWS Chatbot was renamed; console still lives under chatbot docs.
- It forwards **SNS** notifications to Slack / Microsoft Teams / Amazon Chime, can run **AWS CLI** from chat with IAM channel roles and guardrails, and supports custom actions (buttons → Lambda/CLI).

**TEAM:** Step Functions send notifications (email/SNS). Those SNS topics **can** be subscribed into Amazon Q Developer in chat applications so approvers *see* requests in Slack/Teams. That is **notification**, not the TEAM approval UX.

**Chat request/approval sample (separate):** [aws-samples/requesting-just-in-time-privileged-access-using-managed-services](https://github.com/aws-samples/requesting-just-in-time-privileged-access-using-managed-services)

That README, fetched 29 Aug 2026, states explicitly:

- TEAM is the **recommended**, more featureful option.
- The SSM Change Manager sample is simpler (two CloudFormation templates; almost no Lambda in the request path).
- **“This solution supports raising and approving requests via a Chat interface (Slack/Teams) using AWS Chatbot, whereas as of writing, TEAM only supports sending chat notifications.”**
- Web/CLI mode and Chatbot mode **cannot both be enabled**.
- Grant/revoke still uses Identity Center assignments, executed from the **management account** (EventBridge → SSM Automation). That **conflicts** with the “keep humans and custom automation out of the management account” goal unless tightly scoped.
- Optional Control Tower path to email a CloudTrail non-readonly summary after revoke.

**Fit to a previous chat-based approval pattern:**

| Need | TEAM | SSM Change Manager + Q Developer in chat apps | Custom |
|---|---|---|---|
| Chat **notification** of pending request | Yes (SNS → optional Q Developer) | Yes | Yes |
| Chat **request + approve** | **No** (as documented by AWS samples) | **Yes** | Build it |
| Rich eligibility / OU policies / in-app session logs | Yes | Limited | Build it |
| Avoid management-account executor | Yes (delegated admin) | Sample executor is in **management account** | Design choice |
| AWS-operated | No | Change Manager is managed; the sample still must be reviewed | No |

Closest AWS-aligned reconstruction of “request in chat → human approval → automatic time-boxed permission set” is **either** TEAM (approval in TEAM UI, notify in chat) **or** the SSM sample (approval in chat) **or** a partner PAM with a Slack/Teams app. None of these is native Identity Center.

---

## 10. Quotas, limits, and operational caveats

From [IAM Identity Center quotas](https://docs.aws.amazon.com/singlesignon/latest/userguide/limits.html) and TEAM docs:

| Limit | Value | JIT impact |
|---|---|---|
| `CreateAccountAssignment` outstanding async calls | **15, not increasable** | Burst of multi-account “first responder” grants can queue/fail. Design for sequential or modest parallelism. |
| Identity Center APIs collective TPS | 20 (read can be raised) | TEAM list APIs; v1.5.0 cache exists because **Organizations** list APIs are 5 TPS. |
| Permission sets per instance | 3500 (increasable) | Fine |
| Provisioned permission sets per account | 500 (increasable) | Fine |
| Groups assignable to a permission set per account | **100, not increasable** | Prefer assigning **users** for TEAM grants (TEAM already assigns the requester user, not a group). |
| Console assignment batch | 10 accounts at a time | Human standing assignments; TEAM is one account per request. |
| Permission-set session | 1–12 h | Leftover session after revoke. |
| TEAM eligibility max duration | 1–8000 h | Do not allow 8000 h for admin sets; set a low max in TEAM settings. |
| TEAM OU expansion | Direct accounts only | Nested prod OUs need multiple eligibility policies or flattened OUs. |
| CloudTrail Lake | Closed to **new** customers **31 May 2026** | TEAM session-log feature blocked for new Lake signups. |
| Identity Center Region | Historically one Region; multi-Region replication GA **Feb 2026** (constraints: org instance, external IdP, CMK, 17 commercial Regions, up to 6 Regions default) | Reduces but does not eliminate break-glass need. TEAM/Cognito remain regional. |
| Management account assignments | Extra IAM permissions required; delegated admin cannot manage them | TEAM will not JIT the tower account. |

---

## 11. Prerequisites checklist

### Identity and org foundation

- [ ] AWS Organizations with **all features** (not billing-only).
- [ ] **Organization instance** of IAM Identity Center enabled (in the management account; administration **delegated**).
- [ ] Identity source connected (external IdP recommended by SRA). SCIM provisioning healthy.
- [ ] MFA at the IdP (or Identity Center MFA if using Identity Center directory / AWS Managed AD / AD Connector).
- [ ] Control Tower landing zone *or* equivalent (Security OU, Log Archive, Audit/Security Tooling accounts). Do not place TEAM in the management account.
- [ ] Dedicated **identity / IdC delegated-admin account** (no workloads). This will be `TEAM_ACCOUNT`.
- [ ] Permission sets already designed and **provisioned** to target accounts (roles exist before the first JIT grant).
- [ ] No standing assignment of elevated permission sets to humans/groups in production.

### TEAM-specific (if operating TEAM)

- [ ] Identity Center groups for **TEAM admins** and **TEAM auditors** (from IdP, attested).
- [ ] Named CLI profiles for management account and TEAM account.
- [ ] TEAM deployed in the **same Region as Identity Center**.
- [ ] `init.sh` (or equivalent) registering delegated admin for Identity Center, CloudTrail, Account Management.
- [ ] TEAM version **≥ 1.2.2** (CVE-2025-1969); prefer **1.5.0**.
- [ ] CloudTrail: existing org trail to Log Archive. Lake organization EDS **only if the org already has Lake** (new Lake signups closed 31 May 2026).
- [ ] WAF on AppSync; S3 access logs on Amplify bucket; TLS/client policy.
- [ ] Eligibility policies, approver policies, max-duration settings, approval required = on.
- [ ] Elevated permission-set **session duration = 1 hour**.
- [ ] SNS → email and/or Amazon Q Developer in chat applications for notifications.
- [ ] Break-glass path tested independently of TEAM.

### Break-glass (mandatory even if TEAM is perfect)

- [ ] Dedicated emergency account, no workloads, no Identity Center assignments.
- [ ] Direct IdP→IAM SAML **or** few MFA IAM users in a vault that does not depend on the same IdP.
- [ ] Pre-created emergency roles in all member accounts, trust only the emergency account, SCP-locked.
- [ ] EventBridge/Security Hub alerts on any use.
- [ ] Documented declaration process and game-day tests.

---

## 12. Audit and accountability controls

The previous pattern’s accountability model (reason + approval + time box + audit, not standing admin) maps onto AWS as follows.

| Control | Where it lives |
|---|---|
| **Who requested, why, which account/set, how long** | TEAM Requests table (or partner PAM / ITSM). Not in Identity Center natively. |
| **Who approved, when** | TEAM (approver is bound server-side post-1.2.2). CloudTrail on the AppSync/Cognito calls is corroboration, not the business record. |
| **When entitlement existed** | CloudTrail in the delegated-admin/management account: `CreateAccountAssignment` / `DeleteAccountAssignment` (`sso-admin.amazonaws.com`). Query by `principalId`, `permissionSetArn`, `targetId`. |
| **When they actually used it** | Target-account CloudTrail: `AssumeRole` / `ConsoleLogin` on the Identity Center role (`AWSReservedSSO_<PermissionSet>_<hash>`). |
| **What they did** | Target-account management events (always) + selected data events. TEAM Lake viewer if Lake exists; otherwise Athena over org trail. TEAM adds +1 hour after assignment end. |
| **Guardrail bypass attempts** | SCP explicit denies; CloudTrail `AccessDenied`; Config non-compliance; Access Analyzer findings. |
| **Standing privilege drift** | Access Analyzer unused access; periodic review of Identity Center assignments (no elevated set should appear except during an open TEAM window). Config/Security Hub for IAM users appearing in member accounts. |
| **Break-glass use** | Dedicated trail + real-time alert; correlate to an incident ticket; rotate after. |

Suggested Athena / Lake question: *For permission set X in account Y, list assignment create/delete around time T, then all non-read events by that assumed-role session.* Identity Center CloudTrail events include the actor; target-account events include `userIdentity.sessionContext` / `principalId` of the SSO role.

---

## 13. Common objection: “a smaller admin group reduces surface — JIT is ceremony”

**Objection.** Keep `AdministratorAccess` standing on a 5-person admin group. Fewer people, MFA, and a trusted group are enough. JIT adds toil and a new app to hack (see CVE-2025-1969).

**AWS-aligned counterarguments** (from Well-Architected, IAM BPs, Identity Center, TEAM, Organizations — not opinion):

1. **Least privilege is about time and task, not just headcount.** SEC03-BP02’s desired outcome is production access **only for the duration of the task**, then **immediate revoke**. A five-person group with 24/7 `AdministratorAccess` on all prod accounts is five standing admins. Attackers, malware on an admin laptop, and session-token theft all get the full window of the portal session (default 8 hours, configurable up to 90 days) plus the role session (up to 12 hours).
2. **Accountability.** Standing admin cannot answer “why did this principal have `iam:CreateUser` at 02:14?” JIT produces request reason, approver, and a closed time box that can be joined to CloudTrail. That is what auditors ask for; “they’re in the admin group” is not a business reason.
3. **Group membership is a back door.** Identity Center explicitly warns that **IdP group admins** can add users to a group that is assigned to the management account (hence: assign **users**, not groups, there). The same applies to a standing prod-admin group. JIT eligibility can still use groups, but the **assignment** is to the requesting **user** and dies automatically.
4. **SCPs do not save the management account.** If the “small admin group” can stand in the tower account, they are outside every SCP. AWS’s mitigation is **don’t give them standing access**, delegate IdC, and JIT even management-account tasks.
5. **Toil is real; ceremony is the control.** TEAM, a partner, or the SSM sample exists *because* Identity Center will not time-box assignments by itself. The operational cost is the price of removing standing admin. Reduce toil with: read-only standing, narrow elevated sets (so many tasks never need admin), auto-approval only for low-risk eligibility (TEAM supports approval-not-required per policy — use sparingly), and chat *notifications* so approvers don’t live in a new UI.
6. **The TEAM CVE is an argument for patching and/or a vendor PAM, not for standing admin.** A spoofable approval is bad; 24/7 admin keys/sessions on laptops is worse and has no patch. If operating TEAM is unacceptable risk, use a **validated partner** (CyberArk, Okta Access Requests, Apono, Tenable) that AWS has already checked against a common JIT requirement set — still not standing admin.
7. **Break-glass is not a substitute for JIT.** Using emergency IAM users for daily prod changes is a documented anti-pattern (SEC03-BP03). It destroys the audit distinction and normalizes the back door.

---

## 14. What AWS provides vs what still has to be built / operated

### AWS provides (managed)

- IAM Identity Center organization instance, permission sets, provisioning of IAM roles into accounts, access portal, CLI/SDK auth.
- `CreateAccountAssignment` / `DeleteAccountAssignment` and related `sso-admin` APIs.
- Permission-set session duration (1–12 h) and portal session duration.
- Delegated administration of Identity Center.
- Organizations, OUs, SCPs, RCPs, Control Tower controls.
- CloudTrail (trails, optionally Lake for existing customers), Config, IAM Access Analyzer, Security Hub, CloudWatch, EventBridge, SNS.
- Amazon Q Developer in chat applications (notifications + optional CLI in chat).
- STS / IAM roles / permission boundaries.
- Emergency-access **guidance** and the ability to configure direct SAML to IAM.
- Centralized member-account root sessions (`AssumeRoot`) for a short list of root-only tasks.
- A **documented partner program** for JIT (since May 2023).
- Multi-Region Identity Center replication (Feb 2026) for standing entitlements.

### AWS provides as **sample / customer-operated** (not a service)

- **TEAM** (request/approve/schedule/grant/revoke UI + Step Functions).
- **SSM Change Manager JIT sample** (including optional chat approve).
- Various break-glass example repos.

### The customer (or a partner) must still

- Choose TEAM vs partner vs custom vs SSM sample.
- Design permission sets (standing vs elevated) and **stop standing-assigning** elevated sets.
- Design eligibility (who may request which account/set) and approver mapping (peer vs separate team).
- Operate and **patch** TEAM (or pay a partner). Budget for CVE response.
- Decide session durations (elevated = 1 h).
- Wire notifications (SNS → email / Q Developer in chat apps). Chat *approval* is extra engineering or the SSM sample.
- Build “who had what when” queries (Lake if already subscribed; else Athena on the org trail). Enable data events where required.
- SCP-cap even JIT admin; permission-boundary elevated sets if needed.
- Implement and test **break-glass**, including management-account root vault.
- Keep humans out of the management account except named, preferably time-boxed, org tasks.
- Lifecycle: joiner/mover/leaver in the IdP; TEAM eligibility follows groups only if those groups are governed.
- Optionally replace `AdministratorAccess` with Access Analyzer-generated policies after a period of JIT use.

### Suggested target operating model (AWS-aligned, team-ready)

1. **Standing:** `ReadOnly` (or equivalent) on the accounts a group actually supports. Nothing more in production.
2. **Elevated permission sets:** a small catalog (`IncidentRespond`, `PowerUserNoIAM`, maybe `AdministratorAccess` as last resort), session duration 1 hour, provisioned to prod accounts, **zero standing assignments**.
3. **TEAM (or partner)** in a dedicated IdC delegated-admin account. Eligibility by IdP group; approvers by account/OU; max duration measured in hours not days for admin sets; approval required.
4. **Chat:** Q Developer in chat applications for **notifications**; keep approval in TEAM (or accept the SSM sample’s chat approve trade-off).
5. **Management/tower account:** dedicated permission sets, user (not group) assignments, as few as possible, prefer JIT via a process that **does** run from the management account (because TEAM cannot). Treat as rarer than prod JIT.
6. **Break-glass:** emergency account + pre-created roles + alarms. Never used for the first-responder path.
7. **Audit:** org CloudTrail → Log Archive; Athena view joining IdC assignment events to target-account activity; Access Analyzer unused-access on a schedule.

---

## 15. Primary source index

### Identity Center and JIT positioning

- https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html
- https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture-identity-management/workforce-iam-identity-center.html
- https://aws.amazon.com/about-aws/whats-new/2023/05/aws-partners-temporary-elevated-access-capabilities-iam-identity-center/
- https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/howtosessionduration.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/assignusers.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/delegated-admin.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/limits.html
- https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_CreateAccountAssignment.html
- https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_DeleteAccountAssignment.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/configure-abac.html
- https://aws.amazon.com/blogs/security/define-a-custom-session-duration-and-terminate-active-sessions-in-iam-identity-center/
- https://aws.amazon.com/blogs/security/getting-started-with-aws-sso-delegated-administration/

### TEAM

- https://github.com/aws-samples/iam-identity-center-team
- https://aws-samples.github.io/iam-identity-center-team/
- https://aws-samples.github.io/iam-identity-center-team/docs/overview/architecture.html
- https://aws-samples.github.io/iam-identity-center-team/docs/overview/workflow.html
- https://aws-samples.github.io/iam-identity-center-team/docs/overview/policies.html
- https://aws-samples.github.io/iam-identity-center-team/docs/overview/security.html
- https://aws-samples.github.io/iam-identity-center-team/docs/overview/cost.html
- https://aws-samples.github.io/iam-identity-center-team/docs/deployment/prerequisites.html
- https://aws-samples.github.io/iam-identity-center-team/docs/deployment/deployment_process.html
- https://aws.amazon.com/blogs/security/temporary-elevated-access-management-with-iam-identity-center/
- https://aws.amazon.com/blogs/security/managing-temporary-elevated-access-to-your-aws-environment/
- https://github.com/aws-samples/iam-identity-center-team/releases/tag/1.5.0
- https://aws.amazon.com/security/security-bulletins/AWS-2025-004/
- https://github.com/aws-samples/iam-identity-center-team/security/advisories/GHSA-x9xv-r58p-qh86

### Organizations / Control Tower / management account

- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices_mgmt-acct.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html
- https://docs.aws.amazon.com/controltower/latest/userguide/best-practices.html
- https://docs.aws.amazon.com/controltower/latest/userguide/tips-for-admin-setup.html

### IAM / STS / Well-Architected

- https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_request.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-getting-started.html
- https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_permissions_least_privileges.html
- https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_permissions_emergency_process.html

### Break-glass / emergency access

- https://docs.aws.amazon.com/singlesignon/latest/userguide/emergency-access.html
- https://aws.amazon.com/blogs/security/secure-root-user-access-for-member-accounts-in-aws-organizations/

### CloudTrail / Config / ChatOps samples

- https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake.html
- https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html
- https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config.html
- https://docs.aws.amazon.com/chatbot/latest/adminguide/what-is.html
- https://github.com/aws-samples/requesting-just-in-time-privileged-access-using-managed-services

### Adjacent 2025–2026 What’s New (not workforce JIT)

- https://aws.amazon.com/about-aws/whats-new/2025/11/streamline-integration-partner-products-iam-delegation/ (IAM temporary delegation to *products*)
- https://aws.amazon.com/about-aws/whats-new/2026/08/aws-iam-aam/ (account access manager)
- https://aws.amazon.com/about-aws/whats-new/2026/02/aws-iam-identity-center-multi-region-aws-account-access-and-application-deployment/

---

## 16. One-page decision

**Native Identity Center JIT in 2026?** No. Integration + APIs only.

**AWS’s own sample for the approval workflow?** TEAM (self-hosted). Partners if operating a sample as PAM is unacceptable.

**Where to run it?** Dedicated Identity Center delegated-admin account. Not the management/tower account. Not member accounts.

**What changes in member accounts?** Nothing deployed. Assignments appear and disappear on already-provisioned permission-set roles.

**What about the old chat-approval path?** Reconstruct with TEAM UI + chat notifications, or the SSM Change Manager + Amazon Q Developer in chat applications sample. Chat approval is not TEAM and not Identity Center native.

**Standing `first_responder` on all prod accounts?** Conflicts with SEC03-BP02 and Identity Center least-privilege guidance. Replace with eligibility + time-boxed assignment. Keep a 1-hour session duration on the elevated set.

**Break-glass?** Mandatory, separate, pre-created, alarmed, never used for daily elevation. TEAM depends on regional Cognito and Identity Center.

**New CloudTrail Lake?** Not available to new customers after 31 May 2026. Plan Athena on the organization trail for “who had what when.”
