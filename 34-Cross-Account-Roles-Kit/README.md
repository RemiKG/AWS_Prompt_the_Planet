# Keyless Cross-Account Access Kit: IAM Roles with sts:ExternalId and AWS RAM Shares That Stop Confused-Deputy Attacks

Replace every pasted access key with scoped IAM roles, vendor-unique external IDs, and RAM resource shares—third parties and sibling accounts get exactly the access they need, revocable in one API call, for $0 in service cost.

## The Problem

IAM access keys never expire on their own. The key your team pasted into a monitoring vendor's portal in 2023 still works today and appears in zero dashboards. Keys leaked to public GitHub are exploited within minutes, and the January 2023 CircleCI breach forced thousands of teams to rotate every stored secret at once—a copied key is a standing credential.

The deeper trap is the **confused deputy**. Your vendor assumes roles in 4,000 customer accounts using one legitimate set of credentials. An attacker signs up for that same vendor, submits YOUR role ARN as "their" account, and the vendor—the deputy—faithfully assumes your role on the attacker's behalf. No key leaks; the trust policy itself is the hole. AWS's documented fix is one exact pattern—a vendor-generated `sts:ExternalId` condition—that teams either skip or implement wrong: customer-chosen IDs, IDs reused across tenants, or ExternalId bolted onto internal roles where it protects nothing.

Internally, teams duplicate entire VPCs into every new account because nobody told them AWS RAM shares subnets and Transit Gateways across accounts for $0—each duplicated VPC drags its own NAT gateway along at $32.85/month before the first GB of data.

This prompt generates the complete production-ready kit: both trust-policy shapes, Terraform, a vendor runbook, a fail-closed verification script, and a 90-day audit query.

## Who This Is For

- Startups onboarding a third-party SaaS (monitoring, cost tooling, security scanning) that just asked for "an IAM user with these permissions"
- Teams splitting one AWS account into an Organization who need a tooling account to reach production without copying credentials
- Anyone who inherited an environment where "cross-account access" means a shared key in a password vault

## How to Use

1. Open Kiro CLI, Claude Code, or Amazon Q Developer CLI in an empty directory, with AWS credentials for the account that will own the roles.
2. Copy the System Prompt below verbatim into the assistant.
3. Replace the bracketed placeholders: `[PRODUCTION_ACCOUNT_ID]`, `[TOOLING_ACCOUNT_ID]`, `[NETWORK_ACCOUNT_ID]`, `[VENDOR_ACCOUNT_ID]`, `[VENDOR_GENERATED_EXTERNAL_ID]`, `[LOG_BUCKET]`, `[ORG_ID]`.
4. Let the assistant run its read-only verification commands BEFORE it writes any policy—reject output that skips this.
5. Review the two trust policies first. The `Condition` blocks are the security boundary; everything else is plumbing.
6. Run `terraform init && terraform apply`, then execute `verify.sh` from a second CLI profile to prove the ExternalId gate works in both directions.
7. Send `vendor-onboarding.md` to the third party; file `offboarding.md` in your runbook wiki before you need it.

**Prerequisites**

- **Required Access:** Administrator (or IAMFullAccess + AWSResourceAccessManagerFullAccess) in the role-owning account; one-time management-account access to enable RAM org sharing; a second CLI profile for the negative test.
- **Recommended Background:** Reading IAM policy JSON; knowing your organization ID (o-...) and OU structure.
- **Tools Required:** AWS CLI v2, Terraform >= 1.5, AWS Documentation MCP server (aws_read_documentation, aws_search_documentation) so the assistant confirms condition-key semantics against live docs.

**Key Parameters:** ExternalId (vendor-generated UUID, 2-1224 chars, unique per customer), MaxSessionDuration (3600s default, 43200s max, role chaining hard-capped at 3600s), internal trust principal (exact role ARN, never :root), aws:PrincipalOrgID ([ORG_ID]), RAM allow_external_principals (false), audit window (90 days), vendor scope (CloudWatchReadOnlyAccess + one log bucket).

**Troubleshooting:** If verify.sh's positive test fails with AccessDenied even though permissions look right, that's the trust policy: STS evaluates the Condition before any permissions policy, so a missing `--external-id` flag or mismatched value fails at the door—diff the Condition block first.

## System Prompt

```
# Cross-Account Access Design Request — IAM Roles, External IDs, RAM Shares (Zero Access Keys)

## Project Overview
Generate production-ready cross-account access for an AWS Organization with a production
account [PRODUCTION_ACCOUNT_ID], a tooling account [TOOLING_ACCOUNT_ID], a network account
[NETWORK_ACCOUNT_ID], and a third-party monitoring vendor operating from
[VENDOR_ACCOUNT_ID]. Replace every IAM user and long-lived access key with assumable IAM
roles and AWS RAM resource shares. Terraform preferred; emit raw JSON trust policies
alongside the modules.

## Verify Reality First (before generating anything)
1. Run aws sts get-caller-identity and aws configure get region — confirm the working
   account and region. Never fabricate account IDs; keep bracketed placeholders for
   anything unconfirmed.
2. Run aws organizations describe-organization to capture the real o- organization ID
   for aws:PrincipalOrgID. If Organizations is not in use, omit that condition and say so.
3. Run aws accessanalyzer list-analyzers — if no external-access analyzer exists, create
   one (free) as deliverable zero.
4. Confirm current sts:ExternalId semantics with aws_read_documentation before writing
   any trust policy.

## Detailed Requirements

### 1. Confused-Deputy Protection (non-negotiable)
The vendor assumes roles in thousands of customer accounts with one set of legitimate
credentials. An attacker can sign up for that same vendor and submit MY role ARN as
"their" account; the vendor — the confused deputy — would then assume my role on the
attacker's behalf. The ExternalId breaks this: the vendor generates a unique value per
customer and always sends the value bound to the requesting tenant, so a foreign role ARN
paired with a mismatched ExternalId fails at the trust policy. Encode this explanation as
comments in the generated files. The vendor trust policy MUST be exactly this shape:

{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::[VENDOR_ACCOUNT_ID]:root" },
    "Action": "sts:AssumeRole",
    "Condition": { "StringEquals": { "sts:ExternalId": "[VENDOR_GENERATED_EXTERNAL_ID]" } }
  }]
}

Rules: the ExternalId is vendor-generated (never customer-chosen), unique per customer,
2-1224 characters, stored like a secret, never reused across tenants. For AWS service
principals use aws:SourceAccount + aws:SourceArn conditions instead — ExternalId is for
third-party cross-account trust only.

### 2. Internal Cross-Account Roles
- prod-deployer role trusted ONLY by arn:aws:iam::[TOOLING_ACCOUNT_ID]:role/tooling-runner
  — pin the exact role ARN, never :root, for in-house trust; add
  "aws:PrincipalOrgID": "[ORG_ID]" as a second condition.
- MaxSessionDuration 3600 on both roles; comment that role chaining is hard-capped at
  3600 seconds regardless of MaxSessionDuration.
- Vendor permissions: managed policy CloudWatchReadOnlyAccess plus s3:GetObject and
  s3:ListBucket scoped to arn:aws:s3:::[LOG_BUCKET] and /* only. No wildcard Action or
  Resource anywhere else in the kit.

### 3. RAM Shares (network account)
- aws_ram_resource_share "shared-network" with allow_external_principals = false;
  associate two private subnet ARNs and the Transit Gateway; principal = the workloads
  OU ARN, not individual account IDs.
- Include the one-time aws ram enable-sharing-with-aws-organization step, commented that
  it runs from the management account.

### 4. Audit & Cost
- Athena SQL over CloudTrail covering 90 days of AssumeRole events: eventName,
  sourceIPAddress, requestParameters.roleArn, errorCode.
- Everything must cost $0 to run (IAM, STS, and RAM are free); flag any deliverable that
  introduces spend before generating it.

## Deliverables Requested
1. trust-policy-vendor.json and trust-policy-internal.json
2. iam.tf, ram.tf, variables.tf with sane defaults
3. vendor-onboarding.md — includes the exact test command:
   aws sts assume-role --role-arn <ROLE_ARN> --role-session-name vendor-test
   --external-id [VENDOR_GENERATED_EXTERNAL_ID]
4. verify.sh — positive test (with ExternalId, expect credentials with 3600s expiry) AND
   negative test (without it, expect AccessDenied; exit 1 if it unexpectedly succeeds)
5. athena-assumerole-audit.sql
6. offboarding.md — instant revoke: delete the role, or deny sessions issued before now
   via the AWSRevokeOlderSessions inline-policy pattern (Deny * with DateLessThan on
   aws:TokenIssueTime)

## Error Management
- AccessDenied on sts:AssumeRole: diff the trust-policy Condition block FIRST; a missing
  --external-id flag is the cause far more often than the permissions policy.
- RAM principal association FAILED: organization sharing is not enabled; stop and emit
  the enable command instead of retrying.
- Run aws accessanalyzer validate-policy on every generated policy; zero ERROR findings.

Output the complete files without preamble. Acceptance bar: verify.sh passes both tests
on the first run and the entire kit deploys in under 15 minutes.
```

## What You Get

1. **trust-policy-vendor.json** — `:root` vendor principal + the exact `StringEquals` / `sts:ExternalId` condition, confused-deputy explanation preserved as comments
2. **trust-policy-internal.json** — pinned role-ARN principal + `aws:PrincipalOrgID` as a second factor
3. **iam.tf, ram.tf, variables.tf** — `aws_iam_role`, `aws_iam_role_policy`, `aws_ram_resource_share`, `aws_ram_resource_association`, `aws_ram_principal_association`
4. **vendor-onboarding.md** — the exact `aws sts assume-role` test command and the four ExternalId rules (vendor-generated, unique, secret, never reused)
5. **verify.sh** — positive and negative assume-role tests; exits 1 if the role is assumable WITHOUT the ExternalId
6. **athena-assumerole-audit.sql** — every AssumeRole event in 90 days: caller, source IP, target role, errorCode
7. **offboarding.md** — instant-revoke procedure (delete role, or the AWSRevokeOlderSessions deny pattern for live sessions)
8. **README.md** — deploy order, rollback, and a cost table

## Example Output

```
$ ./verify.sh
[1/4] Caller: arn:aws:iam::111122223333:user/admin (us-east-1) ... OK
[2/4] Positive test: assume-role WITH --external-id ............ PASS (expires in 3600s)
[3/4] Negative test: assume-role WITHOUT --external-id ......... PASS (AccessDenied, fail-closed)
[4/4] accessanalyzer validate-policy x4 ........................ PASS (0 ERROR findings)
RAM share "shared-network": ASSOCIATED (2 subnets, 1 transit gateway -> ou-a1b2-workloads)
Kit cost this month: ~$0.05 (CloudTrail S3 storage only)
```

## AWS Services Used

AWS Identity and Access Management (IAM), AWS Security Token Service (STS), AWS Resource Access Manager (RAM), AWS Organizations, IAM Access Analyzer, AWS CloudTrail, Amazon Athena, Amazon S3

## Well-Architected Alignment

- **Security:** Eliminates long-term credentials (SEC02), enforces least privilege (SEC03), implements the confused-deputy mitigation exactly as the IAM documentation prescribes; every policy passes an IAM Access Analyzer `validate-policy` gate.
- **Operational Excellence:** Onboarding/offboarding runbooks exist before the first vendor call; the verification script turns "I think it's secure" into a repeatable check.
- **Reliability:** The negative test proves the gate fails closed; the instant-revoke path is documented before an incident, not during one.
- **Cost Optimization:** The entire access layer runs on free services; RAM sharing removes the reflex to duplicate VPC plumbing per account.

## Cost Notes

- IAM roles and policies: $0. STS AssumeRole calls: $0. AWS RAM: $0.
- CloudTrail: the first copy of management events is free; S3 storage at $0.023/GB-month means 90 days of a small org's events (~2 GB) costs about $0.05/month.
- Athena audit query: $5 per TB scanned; the 90-day query scans well under 1 GB, so under $0.005 per run.
- Optional Access Analyzer unused-access analyzer: $0.20 per IAM role or user per month (50 roles = $10/month); the external-access analyzer this kit uses is free.
- Avoided spend: each VPC you don't duplicate saves a NAT gateway at $32.85/month plus data processing; each key you never issue avoids an incident with unbounded cost.

## Troubleshooting

1. **AccessDenied on sts:AssumeRole though permissions look correct** — Cause: the trust-policy Condition fails first; the caller omitted `--external-id` or sent a mismatched value. Fix: re-run with the exact ExternalId from vendor-onboarding.md; diff the Condition block before touching permissions policies.
2. **RAM principal association FAILED for an OU principal** — Cause: sharing with AWS Organizations is not enabled, so OU ARNs are rejected. Fix: run `aws ram enable-sharing-with-aws-organization` from the management account, then re-run `terraform apply`.
3. **Shared subnets accepted but invisible in the participant account** — Cause: RAM shares are regional. Fix: switch the participant console/CLI to the same region as the resource share.
4. **Sessions die after 1 hour despite MaxSessionDuration 43200** — Cause: STS hard-caps role-chained sessions at 3600 seconds. Fix: assume the target role directly from a base identity, or design for hourly refresh.
5. **Access Analyzer flags the vendor role as external access** — Cause: working as intended; the finding documents deliberate cross-account trust. Fix: archive it with a rule scoped to that role ARN and vendor account so real anomalies stay visible.
