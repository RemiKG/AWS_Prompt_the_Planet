# Zero-SSH AWS: Kill the Bastion with SSM Session Manager, Tag-Scoped IAM, and Full Session Audit to S3

Replace bastion hosts, public IPs, and shared `.pem` files with AWS Systems Manager Session Manager—engineers keep their shells and database tunnels, auditors get per-identity keystroke logs, and port 22 closes across the entire account.

## The Problem

Most teams still reach private EC2 and RDS the 2010 way: a bastion host with a public IP, port 22 open to "the office CIDR" (which quietly becomes 0.0.0.0/0 during one late-night incident), and a `prod.pem` file copied across a dozen laptops. The bill for that convenience:

| | Bastion host (per environment) | Session Manager |
|---|---|---|
| Compute (t3.micro, us-east-1) | $7.59/mo | $0 — free SSM feature |
| Public IPv4 address | $3.65/mo | $0 — no public IPs anywhere |
| EBS root volume (8 GB gp3) | $0.64/mo | $0 |
| VPC interface endpoints | $0 | $0 with existing outbound 443; $21.90/mo (3 endpoints, 1 AZ) or $43.80/mo (2 AZ, recommended) on fully private subnets |
| Inbound security-group rules | tcp/22 from a CIDR | zero inbound rules |
| Credentials that can leak | `.pem` files on every laptop | none — IAM + short-lived STS only |
| Patching and AMI rotation | ~2 engineer-hours/mo | none — no host to patch |
| Who-did-what audit | none (shared key = shared identity) | full keystroke log per IAM identity |
| Offboarding an engineer | rotate keys on every host | deactivate one IAM identity |

Three environments means roughly $36/month of bastion spend plus 6 hours of monthly patching toil—buying you a bigger attack surface and a SOC 2 finding ("shared SSH credentials, no session attribution"). The pieces to fix it are all free; what is missing is the wiring: the session-preferences document, the S3 audit bucket with a real retention policy, and the one IAM policy that scopes exactly who can reach exactly what.

## Who This Is For

Startups heading into their first SOC 2 Type II audit, platform engineers who inherited a bastion nobody dares patch, and DBAs who need a psql tunnel to a private RDS endpoint without anyone provisioning a jump box. Works for 3 instances or 300.

## How to Use

1. Copy the System Prompt below into Claude Code, Kiro CLI, or Amazon Q Developer CLI (`q chat`) in an empty working directory.
2. Replace the bracketed placeholders: `[ACCOUNT_ID]`, `[REGION]`, `[TEAM_TAG_KEY]`/`[TEAM_TAG_VALUE]`, `[RDS_ENDPOINT]`, `[LOG_BUCKET_NAME]`.
3. Let the assistant run its read-only reality checks first (`aws ssm describe-instance-information`, security-group scan for port 22). It must not generate Terraform until it has real instance IDs and agent versions in hand.
4. Review the plan, then `terraform apply` (typically 18 resources, under 5 minutes).
5. Run `./verify.sh` for PASS/FAIL evidence, then follow `decommission-runbook.md` to terminate the bastions, release their Elastic IPs, and delete every port-22 rule.

**Prerequisites**

- **Required Access:** IAM permissions to create roles, policies, SSM documents, S3 buckets, KMS keys, CloudWatch log groups, and VPC endpoints; read access to EC2 and RDS inventory.
- **Recommended Background:** basic IAM policy literacy; comfort reading a `terraform plan`.
- **Tools Required:** AWS CLI v2 with the `session-manager-plugin` installed locally; Terraform >= 1.5; AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) so the assistant can confirm document parameters and condition keys during generation.

**Key Parameters:** idle session timeout (20 min default, 60 max), max session duration (60 min prod, 1440 max), S3 audit retention (Glacier Instant Retrieval at 90d, expire 400d), CloudWatch Logs retention (90d), access tag (`Team=platform`), local forward ports (15432→5432 Postgres, 13306→3306 MySQL, 18080→private HTTP), endpoint AZ count (2).

**Troubleshooting:** If `start-session` returns `TargetNotConnected` on a clearly running instance, it's expected on private subnets with no outbound HTTPS—the agent cannot register with the `ssmmessages` endpoint. Deploy the three interface endpoints (or a NAT route) first, then allow up to 5 minutes for the agent to re-register.

## System Prompt

```
# Zero-SSH Access Migration: Architecture & IaC Request

## Project Overview
Act as a senior AWS security engineer. Replace every SSH path into my AWS account — bastion hosts, public IPs, and shared .pem key pairs — with AWS Systems Manager Session Manager. The result must be production-ready: zero inbound security-group rules, every session attributable to a single IAM identity, keystroke logs retained for audit, and port forwarding that reaches private RDS endpoints with no jump box. Terraform preferred for all infrastructure.

## Verify Reality Before Generating
1. Run aws sts get-caller-identity and aws configure get region. Confirm account [ACCOUNT_ID] and region [REGION]; stop and ask if they differ.
2. Run aws ssm describe-instance-information --query 'InstanceInformationList[].[InstanceId,PingStatus,AgentVersion]'. Report which instances are Online, which are unregistered, and which run SSM Agent older than 3.1.1374.0 (minimum for remote-host port forwarding).
3. Run aws ec2 describe-security-groups --filters Name=ip-permission.from-port,Values=22 to enumerate every group still exposing SSH — these feed the decommission runbook.
4. Check whether an SSM-SessionManagerRunShell document already exists; if so, plan a Terraform import instead of a name collision.
Never invent instance IDs, subnet IDs, or hostnames — use only values returned by these calls.

## Detailed Requirements

### 1. Instance Enrollment
- One IAM instance profile carrying the AWS managed policy AmazonSSMManagedInstanceCore plus an inline policy granting s3:PutObject on the audit prefix, logs:CreateLogStream and logs:PutLogEvents on the session log group, and kms:Decrypt + kms:GenerateDataKey on the session KMS key.
- For private subnets with no outbound 443: interface VPC endpoints for ssm, ssmmessages, and ec2messages across 2 AZs. State the cost (~$0.01/endpoint-hour per AZ, about $43.80/month for the 2-AZ set) so I can choose endpoints versus an existing NAT route.

### 2. Session Logging & Audit
- Customer-managed KMS key encrypting the session data channel end to end.
- S3 audit bucket [LOG_BUCKET_NAME]: versioning enabled, SSE-KMS default encryption, Block Public Access all-on, bucket policy denying non-TLS requests, lifecycle rule transitioning session logs to Glacier Instant Retrieval at 90 days and expiring them at 400 days.
- CloudWatch log group /ssm/sessions with 90-day retention and cloudWatchStreamingEnabled true for near-real-time visibility.
- Session preferences document (SSM-SessionManagerRunShell): idleSessionTimeout 20, maxSessionDuration 60, runAsEnabled false.

### 3. The IAM Policy That Scopes Who Reaches What
- Policy ssm-developer-access: allow ssm:StartSession only on arn:aws:ec2:[REGION]:[ACCOUNT_ID]:instance/* where ssm:resourceTag/[TEAM_TAG_KEY] equals [TEAM_TAG_VALUE]; allow it on exactly two documents — the account-owned SSM-SessionManagerRunShell and the AWS-owned AWS-StartPortForwardingSessionToRemoteHost (note its ARN has an empty account field) — and enforce BoolIfExists ssm:SessionDocumentAccessCheck = true so no other document can ever be smuggled in.
- Restrict ssm:TerminateSession and ssm:ResumeSession to arn:aws:ssm:*:*:session/${aws:username}-* so nobody can kill another engineer's session.
- Add an explicit account-wide Deny on ssm:StartSession with document AWS-StartSSHSession — SSH-over-SSM stays dead too.

### 4. Port Forwarding Toolkit
- connect.sh with copy-paste one-liners: a plain shell session; local 15432 to [RDS_ENDPOINT]:5432 via AWS-StartPortForwardingSessionToRemoteHost; local 13306 to MySQL 3306; local 18080 to any private ALB.
- A security-group rule letting the forwarding instance's SG reach the RDS SG on 5432, referenced by SG ID, never by CIDR.

### 5. Decommission & Cost
- decommission-runbook.md: drain and terminate bastions, release each Elastic IP ($3.65/month), delete every port-22 rule found in reality-check step 3, deregister key pairs.
- cost-comparison.md: bastion versus Session Manager across compute, public IPv4, patching hours, and attack surface.

## Deliverables Requested
1. ssm-core.tf, logging.tf, iam-access.tf, endpoints.tf (count-gated), variables.tf with sane defaults
2. connect.sh and verify.sh
3. decommission-runbook.md and cost-comparison.md

## Verification & Acceptance
verify.sh must emit evidence the system works: start a session against a tagged instance and confirm a log object lands in S3 within 2 minutes; attempt StartSession on an untagged instance and assert AccessDeniedException; open the RDS tunnel and run pg_isready against localhost:15432; confirm zero security groups still permit inbound 22. Print PASS/FAIL per check and exit non-zero on any FAIL.

Align with the Well-Architected Security and Cost Optimization pillars. Output the Terraform first, files clearly separated, without any preamble. The full stack must apply in under 30 minutes and add zero inbound security-group rules.
```

## What You Get

1. **ssm-core.tf** — instance profile with `AmazonSSMManagedInstanceCore`, customer-managed KMS key, and the `SSM-SessionManagerRunShell` preferences document (20-min idle timeout, 60-min max session).
2. **logging.tf** — SSE-KMS S3 audit bucket with the 90d→Glacier IR / 400d-expire lifecycle, plus the `/ssm/sessions` CloudWatch log group at 90-day retention.
3. **iam-access.tf** — the tag-scoped `ssm-developer-access` policy, own-session-only Terminate/Resume, and the explicit `AWS-StartSSHSession` deny.
4. **endpoints.tf** — conditional `ssm`/`ssmmessages`/`ec2messages` interface endpoints for fully private subnets.
5. **connect.sh** — shell-session and RDS/MySQL/ALB port-forwarding one-liners.
6. **verify.sh** — four PASS/FAIL acceptance checks proving logging, scoping, tunneling, and port-22 closure.
7. **decommission-runbook.md** + **cost-comparison.md** — the bastion teardown plan and the dollars it frees.

## Example Output

```
Reality check: 14 instances registered, 12 Online, 2 missing instance profile (i-0f3..., i-09a...)
3 security groups still allow inbound tcp/22 -> queued in decommission-runbook.md

Tunnel to prod Postgres (no bastion, no public IP):
aws ssm start-session --target i-0a1b2c3d4e5f67890 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["prod-db.cluster-abc123xyz.us-east-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["15432"]}'

iam-access.tf (excerpt):
  Condition = { StringEquals = { "ssm:resourceTag/Team" = "platform" },
                BoolIfExists = { "ssm:SessionDocumentAccessCheck" = "true" } }

verify.sh: PASS log object in s3://acme-ssm-audit/sessions/ after 41s
verify.sh: PASS AccessDeniedException on untagged i-0c9d...
verify.sh: PASS localhost:15432 accepting connections
verify.sh: PASS 0 security groups allow inbound 22
```

## AWS Services Used

AWS Systems Manager (Session Manager), AWS Identity and Access Management (IAM), Amazon S3, Amazon CloudWatch Logs, AWS Key Management Service (KMS), Amazon EC2, Amazon VPC (interface endpoints), AWS CloudTrail (StartSession API audit), Amazon RDS (port-forwarding target).

## Well-Architected Alignment

- **Security:** zero inbound rules and zero public IPs collapse the attack surface; tag-scoped `ssm:StartSession` plus `ssm:SessionDocumentAccessCheck` is least privilege by construction; KMS encrypts the session channel; every keystroke is attributable to one IAM principal in S3 and CloudTrail.
- **Cost Optimization:** removes per-environment bastion spend ($11.88/month each in compute + IPv4 + EBS), states the endpoint-versus-NAT trade-off explicitly, and tiers audit logs to Glacier Instant Retrieval at 90 days.
- **Operational Excellence:** no bastion AMI to patch; `verify.sh` turns "trust me" into PASS/FAIL evidence; the decommission runbook makes rollback of the old path deliberate, not accidental.
- **Reliability:** access no longer depends on a single jump host; interface endpoints span 2 AZs.

## Cost Notes

Session Manager itself is free. If instances already have outbound 443 (NAT or IGW), incremental cost is ~$0. Fully private subnets need 3 interface endpoints: $0.01/hour each per AZ = $21.90/month per AZ, $43.80/month for the recommended 2-AZ set, plus $0.01/GB processed. Audit storage is small—interactive session logs rarely exceed 100 MB/month, about $0.05 of CloudWatch ingestion ($0.50/GB) and pennies in S3 ($0.023/GB-month Standard, $0.004 in Glacier IR). Against that: each decommissioned bastion returns $7.59 (t3.micro) + $3.65 (public IPv4) + $0.64 (8 GB gp3) = $11.88/month, and roughly 2 engineer-hours of monthly patching per host.

## Troubleshooting

1. **`TargetNotConnected` on a running instance** — Cause: no path to the SSM endpoints (missing outbound 443 or VPC endpoints) or no instance profile. → Fix: attach `AmazonSSMManagedInstanceCore`, add the three interface endpoints or a NAT route, and wait up to 5 minutes for the agent to register.
2. **`SessionManagerPlugin is not found`** — Cause: the CLI plugin is a separate install from AWS CLI v2. → Fix: install `session-manager-plugin` locally and restart your terminal.
3. **Sessions work but nothing lands in S3** — Cause: the *instance role* (not the user) lacks `s3:PutObject` or `kms:GenerateDataKey` on the audit bucket/key. → Fix: add both to the instance profile's inline policy; the agent uploads the log at session end.
4. **RDS tunnel opens but `psql` hangs** — Cause: the RDS security group doesn't allow 5432 from the forwarding instance's SG, or the agent predates 3.1.1374.0. → Fix: add an SG-ID-referenced ingress rule and update SSM Agent.
5. **`AccessDeniedException` on StartSession despite the new policy** — Cause: tag value case mismatch on `ssm:resourceTag/Team`, or the document isn't listed while `SessionDocumentAccessCheck` is enforced. → Fix: align tag casing exactly and add the document ARN to the allow statement.

Port 22 was never the product. Close it, keep the audit trail, and let offboarding be one IAM call.
