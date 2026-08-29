# IAM Identity Center / just-in-time elevated access

Standing least-privilege plus temporary elevated access for privileged permission sets. As of 2026-08-29, Identity Center has **no native managed JIT**. The grant/revoke mechanism is `CreateAccountAssignment` / `DeleteAccountAssignment`; request/approve/time-box is TEAM, a validated partner, or custom orchestration.

Elevated permission-set **session duration should be 1 hour** — revoke does not kill in-flight sessions. Deploy the workflow in the Identity Center **delegated-admin account**, not the management account. Assignments are **per account**. Break-glass is a separate alarmed path.

## Notes

- [Time-boxed access (team proposal)](team-proposal.md)
- [JIT research brief](jit-research.md) — Identity Center, TEAM, Control Tower, break-glass
- [Teams request/approval](teams-elevation.md) — Microsoft Teams vs TEAM vs Entra PIM vs Amazon Q

Primary AWS references: [Temporary elevated access](https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html) · [TEAM sample](https://github.com/aws-samples/iam-identity-center-team) · [Emergency access](https://docs.aws.amazon.com/singlesignon/latest/userguide/emergency-access.html)
