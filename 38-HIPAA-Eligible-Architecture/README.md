# HIPAA Launch Kit: BAA-Gated, KMS-Encrypted PHI Architecture That Blocks Non-Eligible AWS Services

Accepts the BAA in AWS Artifact, hard-gates every proposed service against the live HIPAA Eligible Services Reference, and ships KMS-encrypted, six-year-audited PHI infrastructure as Terraform — so your first health-system contract survives security diligence instead of dying in it.

## The Problem

Healthtech startups don't fail HIPAA on encryption — they fail it on scoping. AWS offers 200+ services, but only the ~150 on the official HIPAA Eligible Services Reference are covered by the Business Associate Addendum (BAA). AWS's own guidance is explicit: services not on that list may still be used, "provided that the services do not process or store ePHI." One Lightsail instance caching patient records or one App Runner service fronting a PHI API — both absent from the eligible list — and your BAA no longer covers that data path. Nobody warns you; the deploy succeeds.

The stakes are concrete: healthcare has been the most expensive industry for breaches for over a decade, averaging $9.77M per incident (IBM Cost of a Data Breach, 2024), and HIPAA's Security Rule requires 6 years of documentation retention (45 CFR 164.316(b)(2)(i)) — a number almost no default log configuration meets. CloudTrail's console default keeps 90 days of event history; CloudWatch Logs defaults to never-expire at $0.03/GB-month forever or, worse, teams set 30 days and destroy their audit trail.

Generic compliance auditors tell you what's wrong after you've built. This prompt is the build-side complement: it constructs the architecture correctly the first time, with the eligibility check enforced as a hard gate before a single line of Terraform is written, and produces the PHI data-flow map your customer's security team will ask for in diligence.

## Who This Is For

- Healthtech founders and platform engineers shipping their first workload that stores, processes, or transmits ePHI on AWS
- Startups facing a hospital, payer, or pharma security questionnaire that asks for a data-flow diagram and audit-log retention numbers
- Teams that have accepted (or are about to accept) the AWS BAA and need the account to actually match it

## How to Use

1. Save the System Prompt below as `hipaa-launch-kit.md`. For Kiro, place it in `.kiro/steering/`; for Claude Code, paste it directly or reference it with `@hipaa-launch-kit.md`; for Amazon Q Developer CLI, paste into the chat session.
2. Replace every bracketed placeholder `[LIKE_THIS]` — application shape, region, scale, and monthly budget.
3. Enable the AWS Documentation MCP server so the assistant can pull the live eligible-services list and verify regional availability instead of trusting training data.
4. Answer the BAA confirmation question truthfully. If you have not accepted the BAA in AWS Artifact, the assistant outputs the acceptance steps and stops — that is correct behavior.
5. Review `eligibility-gate.md` first. Every row must read PASS before you look at Terraform.
6. Run `terraform init && terraform plan`, review, apply, then execute every command in `verification.md` and keep the output as audit evidence.

**Prerequisites**

- **Required Access:** An AWS account where you can accept agreements in AWS Artifact (IAM action `artifact:AcceptAgreement`; under AWS Organizations, accept the organization BAA from the management account). IAM permissions to create KMS keys, CloudTrail trails, VPC resources, AWS Config recorders, and S3 buckets.
- **Recommended Background:** Basic Terraform, and a working definition of what counts as ePHI in your product (names + health data, MRNs, claims, device readings tied to a person).
- **Tools Required:** Terraform >= 1.5, AWS CLI v2, AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`), Kiro CLI or Claude Code or Amazon Q Developer.

**Key Parameters:** AWS_REGION (us-east-1), PEAK_RPS (50), PHI_RECORDS (100k), MONTHLY_BUDGET ($800 w/ 80%/100% alerts), audit-log retention (2,192 days = 6 years), app-log retention (365d), flow-log expiry (365d), RDS backup retention (35d), KMS rotation (annual), TLS floor (1.2), Glacier Deep Archive transition (365d).

**Troubleshooting:** If the assistant flags a service you rely on as missing from the eligible list — App Runner and Lightsail are the common surprises — that is the gate working as designed. Move PHI off that path or swap the service; never instruct the assistant to override the gate.

## System Prompt

```
# HIPAA-Eligible First Workload — Architecture Design Request

## Project Overview
We are a healthtech startup shipping our first workload that stores, processes, and transmits electronic Protected Health Information (ePHI) on AWS. Build the production-ready architecture and Terraform for it — this is a build, not an audit. Application shape: [APP_SHAPE — e.g., "REST API + React SPA + Postgres + document uploads"]. Region: [AWS_REGION, default us-east-1]. Scale: [PEAK_RPS, default 50] req/sec, [PHI_RECORDS, default 100k] patient records, [MONTHLY_BUDGET, default $800]/month total infrastructure budget.

## Verify Reality Before Generating (hard stops)
1. BAA gate: Ask me to confirm I have accepted the AWS Business Associate Addendum in AWS Artifact (Artifact > Agreements; organization agreement from the management account if using AWS Organizations). If not accepted, output the exact acceptance steps and STOP. No PHI design ships without an executed BAA.
2. Eligible-services gate: Fetch the live list at https://aws.amazon.com/compliance/hipaa-eligible-services-reference/ using aws_read_documentation or web access. Every service that touches PHI MUST appear on that list as of today — do not rely on training data; the list changes. Services not on the list (e.g., Amazon Lightsail, AWS App Runner) are permitted ONLY on paths that can never process or store ePHI, per AWS's published guidance, and you must label those paths explicitly.
3. Confirm every selected service is available in [AWS_REGION] and that default quotas cover the design (e.g., 5 Elastic IPs, 5 VPCs per Region) before writing Terraform. If a check cannot be performed, say so and list it as a manual pre-flight step — never silently assume.

## Detailed Requirements
### 1. Network Isolation
- One VPC (/16) with PHI compute only in private subnets across 2 AZs; zero public IPs on PHI workloads.
- VPC Flow Logs delivered to S3, 365-day expiry.
- Admin access exclusively via SSM Session Manager (no SSH keys, no bastion); session logs to CloudWatch Logs.
- Gateway VPC endpoints for S3 and DynamoDB (free); interface endpoints for KMS and Secrets Manager so PHI control-plane traffic never transits the public internet.

### 2. Encryption Defaults (non-negotiable)
- Three customer-managed KMS keys with automatic annual rotation: alias/phi-data, alias/phi-logs, alias/phi-backups.
- S3: SSE-KMS default encryption with alias/phi-data, account-level Block Public Access, bucket policies that deny aws:SecureTransport=false and unencrypted PutObject.
- RDS PostgreSQL: storage_encrypted=true (alias/phi-data), parameter rds.force_ssl=1, automated backup retention 35 days, Multi-AZ.
- TLS 1.2 minimum end to end; ALB listener policy ELBSecurityPolicy-TLS13-1-2-2021-06.

### 3. Audit Logging & Retention (exact numbers required)
- CloudTrail: one multi-Region trail, log file validation enabled, delivered to a dedicated S3 bucket with Object Lock in governance mode, default retention 2,192 days (6 years — HIPAA documentation retention, 45 CFR 164.316(b)(2)(i)).
- Lifecycle: transition audit logs to S3 Glacier Deep Archive at 365 days; expire at 2,192 days.
- CloudWatch Logs: application log groups at 365-day retention. PHI must never be written to logs — include a log-scrubbing note for the application layer.
- Enable GuardDuty and AWS Config with the "Operational Best Practices for HIPAA Security" conformance pack.

### 4. Access Control
- IAM roles only — no IAM users with long-lived access keys. Least-privilege policies scoped to the three KMS keys. Two named roles: PHIDataAdmin (break-glass, MFA-required) and ReadOnlyAuditor (for customer diligence reviews).

### 5. Cost Discipline
- Security/logging baseline under $75/month before compute. Deliver a per-service monthly cost estimate. AWS Budgets alerts at 80% and 100% of [MONTHLY_BUDGET].

## Deliverables Requested (Terraform preferred, v1.5+)
1. terraform/ — modules: network, kms, data, logging, iam. Must pass terraform validate.
2. phi-data-flow-map.md — every hop PHI takes (client → ALB → app → RDS/S3 → backups → logs), with encryption state, KMS key, and HIPAA-eligibility status at each hop. This is a customer-diligence deliverable; make it presentable.
3. eligibility-gate.md — one table: service | on the live eligible list today (Yes/No) | touches PHI (Yes/No) | verdict. Any "No + Yes" row is a build blocker, not a footnote.
4. baa-checklist.md — Artifact acceptance steps plus the shared-responsibility items AWS does NOT cover: workforce training, access reviews, risk analysis, breach-notification procedures.
5. verification.md — acceptance evidence: aws s3api get-bucket-encryption, aws cloudtrail get-trail-status, aws kms get-key-rotation-status, aws configservice describe-conformance-pack-status, plus one negative test — an unencrypted PutObject that must fail with AccessDenied.

If any hard stop fails, report the failure and the remediation — do not generate around it. Align the design to the Well-Architected Security and Reliability pillars and state where each control maps. Output the files directly, without preamble.
```

## What You Get

1. **`terraform/`** — five plan-clean modules (`network`, `kms`, `data`, `logging`, `iam`) implementing the full baseline: VPC with private-only PHI subnets, three rotating customer-managed KMS keys, encrypted S3 + RDS, multi-Region CloudTrail with Object Lock, GuardDuty, Config conformance pack, and the two named IAM roles.
2. **`phi-data-flow-map.md`** — the diligence artifact: every hop PHI takes, annotated with encryption state, KMS key alias, and eligibility status. Hand this to your customer's security team as-is.
3. **`eligibility-gate.md`** — the hard-gate table proving every PHI-touching service was on the live eligible list on the build date, with the date stamped.
4. **`baa-checklist.md`** — AWS Artifact acceptance steps plus the human-side HIPAA obligations the BAA does not absorb.
5. **`verification.md`** — copy-paste CLI evidence the controls actually hold, including the negative test (unencrypted upload rejected with `AccessDenied`).

## Example Output

```
## eligibility-gate.md (checked 2026-06-10 against the live HIPAA Eligible Services Reference)

| Service              | On eligible list | Touches PHI | Verdict        |
|----------------------|------------------|-------------|----------------|
| Amazon ECS on Fargate| Yes              | Yes         | PASS           |
| Amazon RDS           | Yes              | Yes         | PASS           |
| Amazon S3            | Yes              | Yes         | PASS           |
| AWS App Runner       | No               | Yes         | BUILD BLOCKER  |

BLOCKER: App Runner is not on the eligible list as of today. Replacing with
ECS on Fargate (eligible) behind the existing ALB. Re-run gate: PASS.

$ aws kms get-key-rotation-status --key-id alias/phi-data
{ "KeyRotationEnabled": true }
$ aws s3api put-object --bucket phi-docs-prod --key t.txt --body t.txt
An error occurred (AccessDenied)... explicit deny: unencrypted PutObject  ✓ expected
```

## AWS Services Used

AWS Artifact (BAA), AWS Key Management Service, AWS CloudTrail, Amazon VPC, Amazon S3 (with Object Lock and Glacier Deep Archive), Amazon RDS, Amazon CloudWatch, AWS Config, Amazon GuardDuty, AWS Secrets Manager, AWS Systems Manager Session Manager, AWS IAM, AWS Budgets, Elastic Load Balancing.

## Well-Architected Alignment

- **Security:** Customer-managed KMS keys with annual rotation, TLS 1.2 floor, private-only PHI subnets, Session Manager instead of SSH, least-privilege roles, account-level Block Public Access — data protection and traceability by default.
- **Operational Excellence:** The verification.md acceptance evidence and the dated eligibility gate turn compliance posture into reviewable, repeatable artifacts rather than tribal knowledge.
- **Reliability:** Multi-AZ RDS, 35-day automated backups under a dedicated backup key, and immutable (Object Lock) audit logs that survive operator error.
- **Cost Optimization:** $75/month baseline ceiling, Glacier Deep Archive at $0.00099/GB-month for years 2-6 of audit logs, free gateway endpoints preferred over interface endpoints, AWS Budgets alerts at 80%/100%.

## Cost Notes

Security/logging baseline at startup scale (before application compute):

- KMS: 3 customer-managed keys × $1.00/month = $3.00, plus $0.03 per 10,000 API requests
- CloudTrail: first copy of management events to S3 is free; S3 data events (if enabled on PHI buckets) $0.10 per 100,000 events
- Audit-log storage: a few GB in S3 Standard at $0.023/GB-month, dropping to $0.00099/GB-month in Glacier Deep Archive after day 365 — six years of logs for under $1/month at typical startup volume
- GuardDuty: roughly $5-15/month at startup scale (30-day free trial to measure)
- AWS Config: $0.003 per configuration item recorded plus $0.001 per rule evaluation; the HIPAA conformance pack typically lands at $5-12/month
- Secrets Manager: $0.40 per secret per month (≈$2 for five secrets)
- Interface VPC endpoints: ~$7.30 per endpoint per AZ per month — starting with KMS + Secrets Manager across 2 AZs ≈ $29; gateway endpoints for S3/DynamoDB are free

Realistic baseline total: **$45-70/month**. The compliance delta over a non-HIPAA build is mostly the endpoints and Config — under $50/month to be defensible in diligence.

## Troubleshooting

- **The assistant designs around a service that is not on the eligible list.** Cause: it answered from training data instead of fetching the live reference. Fix: confirm the AWS Documentation MCP server is connected and re-run; the prompt requires the live list with a dated gate table.
- **BAA acceptance option missing in AWS Artifact.** Cause: in AWS Organizations, member accounts inherit the organization agreement — it must be accepted from the management account. Fix: sign in to the management account, Artifact > Agreements > Organization agreements, accept the BAA there.
- **`terraform apply` fails creating the CloudTrail bucket with Object Lock.** Cause: Object Lock can only be enabled at bucket creation (`object_lock_enabled = true`), not retrofitted. Fix: create a new bucket with the flag set; don't try to modify an existing one.
- **Unencrypted-upload negative test unexpectedly succeeds.** Cause: the deny statement evaluates `s3:x-amz-server-side-encryption` only when the header is present, or Block Public Access masked the test path. Fix: use a `Null` condition on the encryption header in the bucket policy and re-test with the verification.md command.
- **Config conformance pack stuck in `CREATE_IN_PROGRESS`.** Cause: the AWS Config recorder isn't running in the region. Fix: `aws configservice describe-configuration-recorder-status`, start the recorder, then re-deploy the pack.

**Bar:** from accepted BAA to a verified, evidence-backed PHI baseline — `terraform apply` plus the full verification.md pass — in under 30 minutes.
