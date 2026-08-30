# Time-boxed AWS access (team proposal)

Standing admin for a handful of people is a *control* model. AWS’s own security guidance is an *accountability* model: least privilege that is standing, plus elevated access that is requested, approved, time-boxed, and logged.

This is not a product you switch on in IAM Identity Center. Identity Center is the federation and permission-set layer. Request / approve / revoke is a workflow you run on top. **Locked engine:** custom Teams bot + Adaptive Cards + Step Functions calling Identity Center `CreateAccountAssignment` / `DeleteAccountAssignment`. Copy TEAM’s state model and APIs; do not ship TEAM as the product. See [Recommended setup](recommended-setup.md).

**As of 29 August 2026.** IAM Identity Center has no native managed JIT feature.

## What we should argue

“Only three people have admin, so the surface is smaller” is true for *standing* privilege and false as a complete control.

- Well-Architected SEC03-BP02: production access for the duration of the task, then revoke. Headcount is not the unit of least privilege. Time and task are.
- A standing admin cannot answer “why did this principal have `AdministratorAccess` at 14:02?” A JIT request can: reason, approver, window, CloudTrail.
- Those three people are a bottleneck *and* a blast radius. A compromised laptop with standing admin is worse than a 4-hour approved elevation that already expired.
- Restricting **dev** admin to three people is the weakest version of this policy. Non-prod should have scoped standing build access. Prod change should be JIT. Admin-everywhere should be almost never.
- Using break-glass (or a human admin) as the daily path is a documented anti-pattern (SEC03-BP03).

We are not proposing to give more people standing admin. We are proposing to **remove standing admin** and put an approval gate in front of a time-boxed permission set.

## Target model

| Access | Standing? | Notes |
| --- | --- | --- |
| Read / observe in prod | Yes, groups | `ReadOnlyAccess` or a custom prod-read set |
| Scoped build in non-prod | Yes, non-prod accounts only | Not `AdministratorAccess` |
| Incident / prod change (`first_responder`, `PowerUser` without IAM) | **No** | Eligible to *request*. Assigned only after approval. |
| `AdministratorAccess` | **No** in member accounts. Never as a broad group in the management account. | Same JIT path, tighter approvers |
| Break-glass | Separate | Direct SAML to emergency IAM roles, or vaulted MFA IAM users. Independent of Identity Center *and* of Teams. Alarmed on every use. |

Example they already know: request `first_responder` for 4 hours on production accounts, with a reason. After approval, Identity Center creates the account assignment. When the window ends, it deletes the assignment.

Two clocks that must be explained, or security will think revoke is instantaneous:

1. **Assignment window** (the 4 hours). After revoke, the permission set disappears from the access portal.
2. **Permission-set session duration** (set this to **1 hour** on elevated sets). An already-issued console/CLI session survives revoke until this expires. Realistic access is `requested duration + session duration`.

Do **not** standing-assign `first_responder` to a group that covers all prod accounts. Provision the role into those accounts. Assign the *person* only after approval. Identity Center assignments are **per account**, not per OU. “All production” means enumerate the prod OU’s accounts (direct children only, unless you recurse yourselves).

## AWS pieces

- **IAM Identity Center** (organization instance) as the only workforce federation point.
- **Permission sets** as the unit of entitlement. Elevated sets: 1-hour session duration.
- **Delegated administrator account** for the JIT workflow. No other workloads in that account. Do not run this in the management account. A delegated-admin workflow **cannot** JIT the management account; keep it that way.
- **Organizations SCPs** on member accounts to cap even JIT admin (lock CloudTrail, forbid leaving the org, forbid creating long-lived IAM users). SCPs do not apply to the management account.
- **CloudTrail** organization trail: `CreateAccountAssignment` / `DeleteAccountAssignment` in the workflow account, plus `AssumeRole` / API activity in the target account on `AWSReservedSSO_*`. Join on a `requestId`. CloudTrail Lake is closed to new customers (31 May 2026); plan Athena on the org trail.
- **IAM Access Analyzer** to catch unused standing access and to shrink anything that is still `AdministratorAccess`.

There is no “block standing admin, allow JIT” SCP switch. The switch is: do not standing-assign elevated permission sets, and only the workflow role may call `CreateAccountAssignment` for those sets.

## Microsoft Teams: three honest options

AWS does not sell “request in Teams, get a time-boxed AWS role.” Ranked against the old Slack pattern:

1. **Custom Teams bot + Adaptive Cards + Step Functions** calling `CreateAccountAssignment` / `DeleteAccountAssignment`, with EventBridge Scheduler to revoke. This is the only path that matches request-in-channel, approve-in-channel, auto-grant, auto-revoke. You own eligibility, identity mapping (Teams UPN → Identity Center user), card refresh, and the bot secret. Highest build cost. Best UX fit.
2. **AWS TEAM** (official sample, latest v1.5.0 as of June 2026). Web app on the Identity Center portal. Step Functions grant/revoke. Eligibility and approver policies, including OU-of-direct-children. Slack notifications are first-class. **Teams is notify-only** (SNS → Lambda → Teams) unless we fork it. Fastest AWS-blessed engine. Must run **≥ 1.2.2** (CVE-2025-1969 spoofed approval). Not a managed service; we patch it.
3. **Entra PIM for Groups + SCIM**, if Entra is already the IdP. Time-boxes *group membership*. Approval is in Entra My Access, not a Teams card. No chat-time picker of “this account + this permission set.” SCIM grant is minutes; **deactivation waits for the incremental cycle**; sessions still linger. Good for a small set of standing *eligible* groups. Poor for a war-room “this account, now.”

**Do not use** Amazon Q Developer in chat applications (formerly AWS Chatbot) as PAM. It can post to Teams and run some CLI. It **hard-denies** `sso:*` and `identitystore:*`. A custom-action Lambda that holds `CreateAccountAssignment` is a confused-deputy, not a product.

## Recommended rollout

**Locked:** custom Teams bot. Chat-native Grant/Deny in a standard `infra` channel **is** the requirement. TEAM cannot do that without a fork. Building the bot against the same Identity Center APIs is v1. Full path: [Recommended setup](recommended-setup.md).

**Before anything else:** ship and *test* break-glass (direct SAML from Entra to a small set of emergency IAM roles in a dedicated emergency account, plus cross-account emergency roles in members). Independent of Identity Center’s Region, independent of Teams, dual-control to retrieve, alarm on use.

**v1:** custom Teams bot in standard channel `infra`. Action message extension dialog (env `dev|uat|prod`, allow-listed permission level, required justification, access duration default 4 hours with a cap). Named approvers get 1:1 Grant/Deny Adaptive Cards. No Grant buttons in the channel. No self-approve. Dev/UAT: one approver. Prod: two distinct approvers. Request expires in 60 minutes if nobody decides. Copy TEAM’s state model and APIs. Stop standing-assigning elevated permission sets. Standing read-only / scoped non-prod remains. Keep Amazon Q off that grant role.

**Do not start with Entra PIM** for this war-room path. Do not deploy TEAM as the product. Do not use Amazon Q Developer in chat applications as PAM.

## Teams channel design

- Dedicated **standard** channel named `infra`. Private channels are a bad fit (Chatbot unsupported; Adaptive Cards unreliable).
- Bot DMs the actionable Approve/Deny card to named approvers; channel gets a redacted summary plus `requestId`.
- Channel membership is visibility, not authorization.
- Update the original card after decision so Approve does not stay clickable.
- System of record is DynamoDB + CloudTrail, not the Teams thread. Justifications in a standard channel are visible to the team; no secrets in cards.
- If Teams is down, that is a break-glass event, not “ping one of the three admins in another app.”

## What we need from the org

- Organization instance of IAM Identity Center, with a delegated administrator account.
- Entra (or other IdP) SAML + SCIM, MFA on the IdP.
- A small permission-set catalog, including `first_responder`, provisioned, **unassigned**.
- Named requester / approver groups (non-prod vs prod).
- Org CloudTrail to the log-archive account.
- Agreement that elevated session duration is 1 hour.
- A tested break-glass path before the bot goes live.

## Suggested decision for the meeting

Adopt the accountability model: no standing elevated assignments, JIT for `first_responder` and admin, break-glass separate.

The engine is locked: **custom Teams bot** for v1. Copy TEAM’s state model and APIs; do not ship TEAM as the UX. Details: [Recommended setup](recommended-setup.md).

Leave Amazon Q in chat applications for ops notifications only (it cannot grant Identity Center permission sets). Leave PIM-for-Groups for later, and only for non-incident eligible groups.

Primary AWS references:

- https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html
- https://docs.aws.amazon.com/wellarchitected/latest/framework/sec_permissions_least_privileges.html
- https://aws.amazon.com/blogs/security/temporary-elevated-access-management-with-iam-identity-center/
- https://github.com/aws-samples/iam-identity-center-team
- https://docs.aws.amazon.com/singlesignon/latest/userguide/emergency-access.html
