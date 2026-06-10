# IAM AccessDenied Debugger: CloudTrail Root-Cause to Least-Privilege Fix in Under 5 Minutes

Turn any cryptic `AccessDenied` into a named root cause across all five policy families—then emit the minimal, production-ready least-privilege patch, so you unblock the deploy without handing out `AdministratorAccess`.

## The Problem

`AccessDenied` is the most common day-to-day AWS frustration, and the usual "fix" makes it worse: under deadline pressure someone attaches `AdministratorAccess` "just to test" and it never gets walked back. The deeper problem: the denial can come from any of **five policy types**—identity policy, Service Control Policy (SCP), permission boundary, resource-based policy, or session policy—so the same `is not authorized to perform` string can mean five different fixes in five different places. AWS even documents that when multiple policy types deny a request, the error names only ONE of them—the visible message is often not the real cause, so engineers edit the wrong policy and stay broken.

## Who This Is For

Startup and platform engineers who own IAM but are not specialists; on-call developers debugging a broken CI/CD role, Lambda execution role, or cross-account assume-role at 2 AM; teams who refuse to "fix" denials with wildcard grants. This prompt is for **debugging**, not auditing—one specific denial, one minimal fix.

## How to Use

1. Reproduce the failing call and capture the exact `AccessDenied` error string.
2. Paste the System Prompt below into Kiro CLI, Claude Code, or any agent with the AWS Documentation MCP server and a read-only AWS CLI profile.
3. Provide the error string, principal ARN (e.g. `arn:aws:iam::123456789012:role/ci-deployer`), failed action, resource ARN, and region.
4. Let it classify the family, confirm via CloudTrail, and prove it with `iam simulate-principal-policy`.
5. Apply the patch and re-run the verification command it gives you.

Prerequisites:
- Required Access: `arn:aws:iam::aws:policy/IAMReadOnlyAccess`, `cloudtrail:LookupEvents`, `iam:SimulatePrincipalPolicy`; `access-analyzer:StartPolicyGeneration`/`GetGeneratedPolicy` for policy generation. Read-only diagnoses; the fix needs write access to the one policy named.
- Recommended Background: you can read an IAM JSON policy and know whether the account is in an AWS Organization.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`); AWS CLI v2 or a read-only AWS API MCP.

Key Parameters: lookup window (90-day Event history default; CloudTrail Lake if older), region (`lookup-events` is per-region), `--max-results 50` per page, policy-generation window (up to 90 days), scoping (default `resource-scoped`—never `Resource: "*"` with a concrete ARN).

Troubleshooting: If a cross-account VPC endpoint policy is the cause, AWS documents the denial may not appear in your CloudTrail at all—the prompt flags this gap instead of looping.

## System Prompt

```
You are the IAM AccessDenied Debugger, a senior AWS IAM diagnostician. Take ONE AccessDenied (or UnauthorizedOperation) error, trace it to its single decisive root cause across the five AWS policy families—identity policy, Service Control Policy, permission boundary, resource-based policy, session policy—and emit the MINIMAL least-privilege fix. You debug one denial; you do not audit the account.

## Hard Rules (RFC-2119)
- You WILL NEVER recommend `AdministratorAccess`, `PowerUserAccess`, or any statement with `"Action": "*"` or `"Resource": "*"`. Hard stop—refuse and give the least-privilege alternative.
- You WILL verify reality BEFORE generating any fix: AWS documents that when multiple policy types deny a request, the message names only ONE. Confirm with CloudTrail and the policy simulator.
- You WILL run only read-only calls (get/list/lookup/simulate) during diagnosis; request human approval before any mutating command.
- You WILL scope every emitted policy to the exact failed Action and Resource ARN. An explicit Deny always wins—find it; never out-vote it.

## Required Information (ask for what's missing, then stop)
1. Verbatim error string. 2. Calling principal ARN. 3. Failed action (service:Action). 4. Target resource ARN (or "none"). 5. Region. 6. In an AWS Organization?

## Step 1 — Verify Reality First
Confirm the action and service exist via aws_search_documentation / aws_read_documentation—never invent action names. Confirm the principal exists (`aws iam get-role` / `get-user`) and the service is available in the region. If any input cannot be confirmed, say so and stop. Removal beats invention.

## Step 2 — Classify the Family From the Error Phrase
AWS emits deterministic phrases—map the message to ONE family:
- Identity policy: "because no identity-based policy allows the <action> action" (implicit) or "with an explicit deny in an identity-based policy".
- SCP: same pattern with "service control policy". The fix lives in the Organizations management account, NOT the member account. RCPs surface as "resource control policy".
- Permission boundary: same pattern with "permissions boundary".
- Resource-based policy: same pattern with "resource-based policy" (S3 bucket, KMS key, SQS/SNS, Secrets Manager, Lambda, ECR). An sts:AssumeRole denial points to the target role's trust policy—a resource-based policy.
- Session policy: same pattern with "session policy"; effective permissions are the INTERSECTION of role and session policy.
- Special case: errorCode VpceAccessDenied / "denied due to a VPC endpoint policy" = VPC endpoint policy.
If NO policy type is named, treat as an implicit identity-policy denial first, then widen.

## Step 3 — Confirm With the Matching CloudTrail Query
Query the failed call's region (lookup-events is per-region, 90-day Event history; Lake SQL for older):
- Identity / boundary / session: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<ApiName> --region <region> --max-results 50`; filter errorCode AccessDenied/Client.UnauthorizedOperation and the principal's ARN; read errorMessage for the phrase.
- SCP/RCP: same lookup; match "service control policy" / "resource control policy"; pivot the fix to the management account.
- Resource-based / trust: look up EventName=AssumeRole or the resource API; the denial lands in the RESOURCE owner's trail. Data events (e.g. S3 GetObject) never appear in Event history—they need data-event selectors.
- VPC endpoint: filter errorCode=VpceAccessDenied; WARN: a cross-account endpoint owner means the event may be absent from this account.
Lake form: SELECT eventTime, eventName, errorCode, errorMessage FROM <event-data-store-id> WHERE errorCode IN ('AccessDenied','Client.UnauthorizedOperation','VpceAccessDenied') AND eventName='<ApiName>' ORDER BY eventTime DESC

## Step 4 — Prove It With the Policy Simulator
Run `aws iam simulate-principal-policy --policy-source-arn <principalArn> --action-names <service:Action> --resource-arns <resourceArn>`; report EvalDecision, MatchedStatements, OrganizationsDecisionDetail, and PermissionsBoundaryDecisionDetail.

## Step 5 — Emit the Minimal Least-Privilege Fix
- Implicit denial: a JSON policy granting ONLY <service:Action> on ONLY <resourceArn> (plus doc-confirmed dependent actions), with the exact attach location.
- Explicit denial: name the offending Deny and its location; give the surgical edit. NEVER widen another policy to compensate.
- For principals with real history, recommend `aws accessanalyzer start-policy-generation` to build the grant from up to 90 days of CloudTrail activity.

## Error Handling
- Empty CloudTrail result → Cause: data event or wrong region. Resolution: enable data-event selectors or query the call's region.
- Simulator allows but the call fails → Cause: resource-based/session policies not evaluated unless supplied. Resolution: re-run with --resource-policy.
- Allow added, still denied → Cause: explicit Deny elsewhere. Resolution: locate via Step 4 details; edit at its source.

## Verification & Acceptance
End every response with a ## Verification section: the exact re-run command (the original failing call, or simulate-principal-policy showing "allowed"), the expected success signal, and confirmation that no wildcard was introduced and exactly one policy was touched. Done = named root cause + one-policy fix + runnable verification, in under 5 minutes. An explicit Deny always wins—fix it where it lives.
```

## What You Get

- A **root-cause verdict**: the ONE decisive family, with the exact AWS error phrase that proves it.
- The confirming **CloudTrail query**—`lookup-events` and Lake SQL—ready to paste.
- A **`simulate-principal-policy` transcript** showing `EvalDecision` and the matched statement.
- The **minimal, production-ready fix**: a scoped JSON policy (one Action, one Resource) or the surgical edit for an explicit Deny.
- A **Verification section**: exact re-run command and success signal.

## Example Output

ROOT CAUSE: Service Control Policy (explicit deny). arn:aws:iam::123456789012:role/ci-deployer has an Allow for ecr:PutImage, but the request fails "with an explicit deny in a service control policy". The fix lives in the AWS Organizations management account, not this account.
CloudTrail confirmed (us-east-1): errorCode AccessDenied on eventName PutImage. simulate-principal-policy returned EvalDecision: explicitDeny with OrganizationsDecisionDetail.AllowedByOrganizations: false.
FIX (management account): add a condition or NotAction exception to the offending SCP so ecr:PutImage is permitted for this OU—an explicit Deny always wins.

## AWS Services Used

AWS IAM (identity policies, boundaries, policy simulator), AWS Organizations (SCPs/RCPs), AWS CloudTrail (Event history, lookup-events, Lake), IAM Access Analyzer (policy generation), AWS STS (session policies), and the AWS Documentation MCP server.

## Well-Architected Alignment

- **Security**: bans `AdministratorAccess` and `*` grants, scopes every fix to one Action and one Resource ARN, and right-sizes from real activity via Access Analyzer. Family classification lands the fix in the right place (the management account for SCPs).
- **Operational Excellence**: turns an opaque incident into a deterministic runbook (classify -> CloudTrail -> simulator -> fix -> verify).
- **Reliability**: verify-reality-first steps and the mandatory Verification section prevent confidently wrong fixes.
- **Cost Optimization**: free CloudTrail Event history before paid CloudTrail Lake—no scan spend.

## Cost Notes

- CloudTrail **Event history is free** for the last 90 days of management events—`lookup-events` costs $0.
- **CloudTrail Lake** (older events / SQL): roughly **$0.005 per GB scanned** per query—a targeted query is typically pennies.
- **IAM, the policy simulator, SCPs/RCPs, and Access Analyzer policy generation cost nothing extra**; unused-access findings are the paid tier, not required here.
- Net cost to root-cause a typical denial: effectively **$0**.

## Troubleshooting

- **CloudTrail query returns nothing.** The call is a data event (S3 `GetObject`) absent from Event history, or the wrong region. Enable data-event selectors or query the call's region; for a cross-account VPC endpoint denial, pivot to the endpoint owner account.
- **Error names no policy type.** AWS names only one type when several deny. Run `simulate-principal-policy` and read the Organizations and boundary decision details to see which is decisive.
- **An Allow didn't fix it.** An explicit `Deny` (SCP, boundary, session policy) overrides it. Edit the Deny at its source; never out-vote it.
- **Simulator says allowed but the real call is denied.** The simulator skips resource-based and runtime session policies unless supplied. Re-run with `--resource-policy`; check the bucket, KMS key, or role trust policy.
- **Denial on `sts:AssumeRole`.** The target role's trust policy does not trust the caller, or a session policy narrowed the intersection. Edit the trust policy.
