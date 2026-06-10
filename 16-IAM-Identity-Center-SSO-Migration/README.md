# IAM Users to Identity Center: Zero-Lockout Workforce SSO Migration with Permission-Set Parity and a Break-Glass Escape Hatch

Move every human off long-lived IAM access keys onto SCIM-provisioned, MFA-enforced SSO with permission sets mapped from your existing IAM policies—so you kill standing credentials without locking yourself out of your own Organization.

## The Problem

Most startups grow up on IAM users: every engineer holds a console password and access keys (`AKIA...`) that never expire, sitting in `~/.aws/credentials` for years. One leaked key can mean a five-figure crypto-mining bill overnight. The fix is IAM Identity Center: SSO from your existing IdP, SCIM deprovisioning the moment HR offboards someone, mandatory MFA, and short-lived role sessions.

But the migration is where teams get burned: (1) **Access regression**—the new permission set silently drops a permission the old inline policy had, breaking a production pipeline. (2) **Total lockout**—the IAM users get deleted before SSO works, and nobody can reach the management account. (3) **Big-bang cutover**—40 engineers switch Monday 9 AM with no rollback. This prompt forces a phased, parity-checked, production-ready migration with a break-glass account kept OUTSIDE SSO in the management account—where SCPs cannot reach—so the escape hatch always works.

## Who This Is For

Startup platform/security engineers running a multi-account AWS Organization with IAM users and long-lived keys, and teams adopting Okta/Entra/Google Workspace wanting HR-driven joiner-mover-leaver automation. This is a **migration runbook generator**, not greenfield SSO setup—existing IAM users and policies must survive at parity.

## How to Use

1. Inventory IAM principals (`aws iam list-users`, `list-attached-user-policies`, `list-groups`) and keep the policy JSON ready.
2. Paste the System Prompt into Kiro CLI, Claude Code, or any agent wired to the AWS Documentation MCP server and a read-only AWS CLI profile.
3. Provide your identity source, Organization layout, and the IAM user/group to job-role mapping.
4. Execute one phase at a time—break-glass first—run the access-parity check after each wave, and delete old IAM users only after the final gate passes.

Prerequisites:
- Required Access: management-account access with `arn:aws:iam::aws:policy/AWSSSOMasterAccountAdministrator`, `AWSSSODirectoryAdministrator`, and `IAMReadOnlyAccess`; AWS Organizations enabled with all features.
- Recommended Background: you can read an IAM JSON policy; permission sets provision into accounts as `AWSReservedSSO_<PermissionSet>_<hash>` roles.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`); AWS CLI v2 for `aws iam list-*` and `aws sso-admin list-permission-sets`.

Key Parameters: permission-set session (PT1H default, PT12H max), portal session (8h default, 15m-90d), break-glass (`BreakGlassAdmin`, hardware MFA, sealed 40+ char password, outside SSO in the management account), SCIM token (1-year expiry, rotate at the 90-day warning), parity (ZERO regressions), waves (2-user pilot -> 25% -> 100%), MFA (always-on, WebAuthn/TOTP), rollback (14 days disabled-not-deleted).

Troubleshooting: Enabling Identity Center but assigning nobody to the management account before disabling IAM console access locks you out—exactly why break-glass is created and TESTED first.

## System Prompt

```
# IAM Users to Identity Center Migration — Requirements Document

You are a senior AWS identity engineer. Generate a production-ready, phased runbook migrating a workforce off long-lived IAM users onto AWS IAM Identity Center, with permission-set parity and a tested break-glass escape hatch. Output Terraform (AWS CLI fallback) plus a step-by-step cutover plan. This is a migration, not greenfield: existing access MUST be preserved at parity.

## Verify Reality FIRST
- Confirm every API name, managed-policy ARN, and SCIM behavior via the AWS Documentation MCP server (aws_read_documentation, aws_search_documentation); never invent either.
- Confirm AWS Organizations has all features enabled and the home Region—Identity Center runs in one Region per Organization.
- Read ACTUAL state with read-only calls (aws iam list-users, list-attached-user-policies, get-user-policy, aws organizations list-accounts); never assume a policy—fetch the JSON.
- If any required input is missing, ASK and STOP. Removal beats invention.

## Hard Rules (RFC-2119)
- You WILL create and TEST break-glass BEFORE touching any IAM user: IAM user BreakGlassAdmin in the management account—where SCPs can never apply—with arn:aws:iam::aws:policy/AdministratorAccess, a sealed 40+ char random password, hardware FIDO2 MFA, NO access keys, NO Identity Center assignment. No IAM user is deleted until break-glass login is proven—a hard stop.
- You WILL achieve ZERO access regression: every user/group's policies map to a superset-or-equal permission set. If parity cannot be proven, FLAG the gap—NEVER silently drop a permission or widen to AdministratorAccess.
- You WILL enforce MFA always-on (WebAuthn/TOTP only) and NEVER keep long-lived keys "as a fallback."
- You WILL phase the cutover—pilot (2 users) -> 25% -> 100%, parity gate between waves—never big-bang.
- You WILL disable, not delete: aws iam update-access-key --status Inactive, hold 14 days, then delete.

## Required Information (ask, then stop if missing)
(1) Identity source: Okta / Entra ID / Google Workspace / Ping / Identity Center directory; (2) account count, OUs, management account ID; (3) IAM user/group -> job-role mapping; (4) home Region.

## Phase 0 — Break-glass (always first)
Create BreakGlassAdmin per the hard rule; PROVE console login and MFA; document its runbook; only then proceed.

## Phase 1 — Identity Center + identity source
Enable Identity Center in the home Region; connect the IdP via SAML 2.0; enable SCIM (token expires after 1 year—rotate at the 90-day warning). MFA always-on; portal session stays at the 8-hour default.

## Phase 2 — Permission-set parity mapping
Per user/group: reuse the same managed-policy ARNs plus an inline policy superset-or-equal of old grants; session PT1H (max PT12H); tag Owner/Department. Emit a parity matrix: OLD vs NEW actions, every delta flagged.

## Phase 3 — Assign + SCIM
Bind permission sets to IdP groups across accounts/OUs (aws sso-admin create-account-assignment). Portal: https://d-xxxxxxxxxx.awsapps.com/start; roles provision as AWSReservedSSO_<PermissionSet>_<hash>.

## Phase 4 — Phased cutover + access-parity check
Per wave, emit an ACCESS-PARITY CHECK: each user runs aws configure sso, aws sso login, aws sts get-caller-identity (must show the AWSReservedSSO_ role); aws iam simulate-principal-policy on OLD principal vs NEW role for the same critical actions (both must return allowed); one real workflow (a CI deploy) succeeds via SSO. Advance only on zero regressions.

## Phase 5 — Decommission
Deactivate every human key, disable console passwords, hold 14 days, delete. Attach an SCP to member-account OUs denying iam:CreateUser/iam:CreateAccessKey for humans; BreakGlassAdmin sits in the management account, untouchable by SCPs.

## Error Handling
- SCIM sync stops: Cause—token expired (1-year life). Resolution—new token, update the IdP, schedule rotation.
- AccessDenied on a known-good action: Cause—parity gap. Resolution—simulate-principal-policy old vs new; add the one missing scoped Allow.
- Management account unreachable: Cause—IAM access removed before an admin assignment existed. Resolution—log in as BreakGlassAdmin, assign an admin permission set, re-test.

## Verification & Acceptance
End with a ## Verification section: (1) break-glass login proof from Phase 0; (2) the parity matrix with ZERO regressions per role; (3) the exact simulate-principal-policy / get-caller-identity commands and expected allowed / AWSReservedSSO_ output; (4) evidence no human long-lived key is active and BreakGlassAdmin has no Identity Center assignment. Done means break-glass proven, parity zero-regression, every human key inactive—target: a fully migrated team in under one business day.
```

## What You Get

- A **Phase 0 break-glass runbook**: create and TEST `BreakGlassAdmin` (AdministratorAccess, hardware FIDO2 MFA, no keys, sealed password) in the management account—outside SSO, beyond any SCP.
- **Terraform (or CLI)** for Identity Center, SAML 2.0 IdP wiring, and SCIM—token-rotation reminder baked in.
- **Parity-mapped permission sets** per job role—same managed-policy ARNs plus superset-or-equal inline policies, PT1H sessions, assigned across accounts/OUs as `AWSReservedSSO_*` roles—plus a **parity matrix** flagging every delta.
- A **phased cutover plan** (2-user pilot -> 25% -> 100%) with parity checks from `aws sso login`, `get-caller-identity`, and `simulate-principal-policy`.
- A **decommission plan**: deactivate keys, disable 14 days, delete, plus an SCP blocking human `iam:CreateAccessKey`/`iam:CreateUser`.

## Example Output

PERMISSION-SET PARITY (DeveloperAccess, source group dev-team, 6 users)
OLD: AWS managed PowerUserAccess + inline allow ssm:StartSession on i-* in us-east-1.
NEW: DeveloperAccess = PowerUserAccess (same ARN) + inline {ssm:StartSession, ssm:TerminateSession on arn:aws:ec2:us-east-1:*:instance/*}. Session PT1H.
PARITY: superset-or-equal, ZERO regressions—simulate-principal-policy returned allowed on old and new. Assigned to Dev (111111111111) and Staging (222222222222).
NEXT GATE: wave 2 waits until all 6 pilot users see AWSReservedSSO_DeveloperAccess_* in `aws sts get-caller-identity` and CI deploys succeed.

## AWS Services Used

AWS IAM Identity Center (permission sets, account assignments, SCIM, MFA, access portal), AWS IAM (existing users/policies, BreakGlassAdmin, SimulatePrincipalPolicy), AWS Organizations (SCPs, OUs), AWS STS (GetCallerIdentity), an external IdP over SAML 2.0 + SCIM (Okta / Entra ID / Google Workspace / Ping), and the AWS Documentation MCP server.

## Well-Architected Alignment

- **Security**: replaces long-lived access keys with short-lived role sessions, always-on phishing-resistant MFA, SCIM leaver deprovisioning, least-privilege permission sets, and an SCP blocking human key creation.
- **Reliability**: the tested break-glass user in the management account—immune to SCPs by design—defends against total lockout; the 14-day disable-not-delete window keeps any cutover problem recoverable.
- **Operational Excellence**: a high-risk one-shot migration becomes a phased, gated runbook with evidence at every step.
- **Cost Optimization**: Identity Center and SCIM cost nothing extra; parity mapping avoids over-provisioning broad admin.

## Cost Notes

- **AWS IAM Identity Center is free**—no charge for permission sets, account assignments, the access portal, or SCIM; IAM, the policy simulator, Organizations, SCPs, and STS add nothing either. You only pay your external IdP per its own pricing.
- Hardware MFA: a FIDO2 key runs roughly $25-$50—buy two so break-glass has no single key of failure.
- Net AWS cost to retire every long-lived key: effectively **$0**.

## Troubleshooting

- **Nobody can log in after disabling IAM users.** Cause: no one was assigned to the management account first. Fix: log in as `BreakGlassAdmin`, assign an admin permission set, re-test before disabling anything.
- **A migrated engineer can do less than before.** Cause: parity gap—the new permission set dropped an old inline grant. Fix: run `simulate-principal-policy` for the failing action against old and new; add the missing scoped Allow, never widen to admin.
- **SCIM silently stops syncing.** Cause: the SCIM token expired (1-year life). Fix: issue a new token in Identity Center, update the IdP, rotate at the 90-day warning.
- **An SCP appears to block break-glass.** Cause: it was created in a member account, where SCPs apply. Fix: SCPs never affect the management account—recreate `BreakGlassAdmin` there and re-test login.
