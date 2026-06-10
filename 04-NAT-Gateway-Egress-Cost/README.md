# NAT-Gateway Zero: Endpoint-First VPC Egress Cost Killer

Attribute every dollar of NAT Gateway traffic to the AWS service that caused it, then replace it with production-ready VPC endpoints—$0.045/GB bill shock becomes $0.01/GB or free, with flow-log proof.

## The Problem

NAT Gateway is the #1 silent line item on early-stage AWS bills: **$0.045/hour = $32.85/month per gateway just to exist**, plus **$0.045 per GB processed**—a 3-AZ VPC pays **$98.55/month idle** before a single byte moves. Most of that traffic is private-subnet instances calling AWS services—S3, ECR, CloudWatch Logs, DynamoDB, Secrets Manager—that never needed to leave the AWS network.

The kicker: **a Gateway VPC endpoint for S3 or DynamoDB is free**, and an Interface VPC endpoint (PrivateLink) costs **$0.01/hour per AZ + $0.01/GB**. 700 GB/month of ECR pulls and log pushes costs $31.50 through NAT, $7.00 through Interface endpoints, and the S3/DynamoDB share goes to $0.

The hard part is **proving which traffic to move**—without VPC Flow Logs you are guessing. This prompt makes the AI write the Athena query that attributes NAT bytes to their destination AWS service, then generates the endpoint-first Terraform that kills the offenders.

## Who This Is For

- Founders who saw "EC2-Other" or "NAT Gateway" spike on Cost Explorer
- Platform teams running multi-AZ VPCs who suspect egress over-spend
- Anyone designing a new VPC who wants endpoint-first networking from day one
- FinOps reviewers who need destination-attributed evidence before approving changes

## How to Use

1. Open Kiro CLI, Claude Code, or any AI assistant with the AWS Documentation MCP server and AWS API access configured.
2. Paste the System Prompt below, replacing `[VPC_ID]`, `[AWS_REGION]`, `[FLOW_LOG_S3_BUCKET]` (or "none yet"), and `[MONTHLY_NAT_BUDGET_USD]`.
3. Let the assistant verify reality first—it confirms the VPC, NAT Gateways, route tables, and Flow Log format before generating.
4. Review the Athena query and ranked offender table.
5. Apply the Terraform, re-run the query after 24-48 hours, and confirm NAT bytes dropped.

Prerequisites:
- Required Access: the AWS managed policy `ReadOnlyAccess` covers discovery; writes need `ec2:CreateVpcEndpoint`, `ec2:CreateFlowLogs`, `ec2:CreateSecurityGroup`, `athena:StartQueryExecution`, `glue:CreateTable`, and `budgets:ModifyBudget` via a least-privilege deployment role.
- Recommended Background: basic VPC subnet/route-table model; comfort reading Terraform and Athena SQL.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`), AWS CLI v2 or an AWS API MCP tool, Terraform >= 1.5, Athena.

Key Parameters: Flow Logs retention via S3 lifecycle (30d dev / 90d prod), aggregation interval (600s default—keep it), traffic type ALL, NAT budget alerts at 80%/100%, Interface endpoint AZ count (match private-subnet AZs, default 2), Gateway endpoints (S3 + DynamoDB, free, always-on), Interface endpoints only when an offender clears break-even (~208 GB/mo per AZ).

Troubleshooting: If the Athena query returns zero rows, Flow Logs use the **default log format**, which omits `pkt-dstaddr` and `flow-direction`—expected; the prompt generates the corrected custom-format definition.

## System Prompt

```
# VPC Egress Cost Architecture Request: NAT-Gateway Zero

## Role
You are a senior AWS network and FinOps engineer. Produce a production-ready, endpoint-first egress design that attributes every byte of NAT Gateway traffic to its destination AWS service, then replaces it with VPC endpoints. Be exact, opinionated, and cost-conscious. No preamble.

## My Environment
- VPC: [VPC_ID]
- Region: [AWS_REGION]
- Flow Logs bucket: [FLOW_LOG_S3_BUCKET] (or "none yet")
- Target monthly NAT spend: [MONTHLY_NAT_BUDGET_USD]

## Step 0 — Verify Reality FIRST
1. Confirm [VPC_ID] exists via describe-subnets, describe-route-tables, and describe-nat-gateways filtered on vpc-id=[VPC_ID]; record each NAT Gateway's ENI ID. If absent, STOP and report.
2. Run describe-vpc-endpoints so you never propose a duplicate.
3. Check Flow Log status and format. If DEFAULT, state that destination attribution requires a CUSTOM format capturing pkt-srcaddr, pkt-dstaddr, flow-direction, bytes, and interface-id, and generate that definition (600s aggregation, to S3).
4. Use aws_read_documentation to confirm CURRENT pricing and that each endpoint service (com.amazonaws.[AWS_REGION].s3, .dynamodb, .ecr.dkr, .ecr.api, .logs, .monitoring, .secretsmanager, .sts, .kms) is AVAILABLE in [AWS_REGION]; cross-check with describe-vpc-endpoint-services.
Never fabricate resource IDs, ARNs, or prices. If a value cannot be verified, say so and use the safe generic form.

## Step 1 — Attribute NAT Traffic by Destination Service
Generate one Athena SQL query over the Flow Logs table that:
- Filters to egress flows through the NAT ENIs from Step 0, grouped by pkt-dstaddr.
- Resolves destinations to the owning AWS service via ip-ranges.json (service codes S3, DYNAMODB, EC2, AMAZON), SUMming bytes per service.
- Outputs: destination_service, total_gb, estimated_monthly_nat_cost_usd (bytes/1073741824 * 0.045), recommended_endpoint.
Also generate the Glue/Athena CREATE TABLE DDL matching the custom format, with partition projection by date.

## Step 2 — Endpoint-First Replacement (Terraform >= 1.5)
For every offender above break-even, generate Terraform:
- aws_vpc_endpoint type "Gateway" for S3 and DynamoDB—FREE; always include both, associating every private route table via route_table_ids.
- aws_vpc_endpoint type "Interface" (PrivateLink) per interface offender (ecr.dkr, ecr.api, logs, monitoring, secretsmanager, sts, kms as needed), private_dns_enabled = true, across the matching private-subnet AZs.
- A dedicated aws_security_group allowing 443 only from the VPC CIDR.
- An AWS Budgets cost budget scoped to NAT Gateway usage (NatGateway-Hours, NatGateway-Bytes) alerting at 80% and 100% of [MONTHLY_NAT_BUDGET_USD].
- State whether any NAT Gateway can be retired outright—a workload that talks only to AWS services needs no NAT.

## Cost Math (show it)
- NAT Gateway: $0.045/hr = $32.85/mo each (730 hrs) + $0.045/GB processed.
- Gateway endpoint (S3/DynamoDB): $0.00.
- Interface endpoint: $0.01/hr per AZ + $0.01/GB (first 1 PB tier).
- Break-even: an interface endpoint beats NAT above (0.01 * 730 * AZ_count)/(0.045 - 0.01) GB/mo, roughly 208 GB per AZ. Show per-offender break-even and projected saving.

## Error Management
- Default flow-log format. Cause: pkt-dstaddr/flow-direction missing. Resolution: emit the custom-format definition, wait one interval, proceed.
- Endpoint service unavailable in [AWS_REGION]. Resolution: keep that service on NAT and flag it—never invent an endpoint name.
- private_dns_enabled rejected. Cause: enableDnsSupport/enableDnsHostnames off. Resolution: emit the VPC attribute fixes first.
- Large unresolved ranges. Resolution: label EXTERNAL (third-party APIs, registries); consolidate onto one shared NAT—never claim endpoint coverage.
Never silently skip a failed check—report Cause and Resolution inline.

## Required Output — in this order
1. Reality-check summary (what exists, flow-log format).
2. Athena DDL + destination-attribution query, copy-paste ready.
3. Ranked offender table with current cost and recommended endpoint.
4. Terraform: endpoints, SG, route-table associations, NAT budget.
5. Before/after monthly cost table with total projected saving.

## Verification & Acceptance
- Exact CLI commands proving endpoints reach "available" and private route tables carry the Gateway prefix-list routes.
- Re-run the Step 1 query 24-48 hours after apply; AWS/NATGateway BytesOutToDestination must trend down and NAT bytes to migrated services must approach 0.
- Acceptance bar: top NAT cost driver identified in under 5 minutes, fix applied in under 15 minutes, NAT data-processing spend cut by at least 60%.
```

## What You Get

- Reality-check summary (VPC, subnets, NAT Gateways, existing endpoints, Flow Log format) plus a corrected flow-log definition when needed
- Glue/Athena `CREATE TABLE` DDL with partition projection, plus the destination-attribution query mapping NAT bytes to services via `ip-ranges.json`
- Ranked offender table: destination_service, total_gb, estimated_monthly_nat_cost_usd, recommended_endpoint
- Terraform: free Gateway endpoints (S3 + DynamoDB), Interface endpoints (ECR, Logs, STS, Secrets Manager, KMS as needed), 443-only security group, route-table associations, AWS Budgets NAT budget at 80%/100%
- Before/after cost table with break-even math, CLI verification commands, and a 24-48h re-measurement plan

## Example Output

Ranked NAT offenders (us-east-1, last 30 days):

| destination_service | total_gb | est_monthly_nat_usd | recommended_endpoint |
| --- | --- | --- | --- |
| EC2 (ECR image pulls) | 512.0 | $23.04 | Interface: ecr.dkr + ecr.api |
| CLOUDWATCH (logs) | 188.4 | $8.48 | Interface: logs |
| S3 | 140.7 | $6.33 | Gateway (FREE) |
| DYNAMODB | 22.1 | $0.99 | Gateway (FREE) |

Before: 3 NAT Gateways = $98.55 fixed + $38.84 data = **$137.39/mo**. After: S3 + DynamoDB move to FREE Gateway endpoints, ECR + Logs (700.4 GB) move to 3 Interface endpoints across 2 AZs ($43.80 hourly + $7.00 data), one NAT retires. After = **~$116.50/mo**, saving **~$21/mo** with NAT data-processing for migrated services at ~$0—and every future GB of growth costs $0.01 instead of $0.045.

## AWS Services Used

Amazon VPC, VPC Endpoints (Gateway + Interface), AWS PrivateLink, NAT Gateway, VPC Flow Logs, Amazon S3, Amazon Athena, AWS Glue, Amazon ECR, Amazon CloudWatch Logs, Amazon DynamoDB, AWS Secrets Manager, AWS STS, AWS KMS, AWS Budgets, IAM.

## Well-Architected Alignment

- **Cost Optimization**: Free Gateway endpoints and $0.01/GB Interface endpoints replace $0.045/GB NAT processing; break-even is computed before any new spend.
- **Security**: AWS-bound traffic stays on the AWS private network; the endpoint security group allows 443 only from the VPC CIDR.
- **Operational Excellence**: Flow Logs + Athena give destination-attributed, queryable evidence; the design ships with a re-measurement acceptance test.
- **Reliability**: Interface endpoints span the same AZs as the private subnets; Gateway endpoints attach to all private route tables.
- **Performance Efficiency**: Private DNS keeps SDK calls unchanged while shortening the path to AWS services.

## Cost Notes

- NAT Gateway: **$0.045/hour ($32.85/month) + $0.045/GB** processed; a 3-AZ fleet is **$98.55/month before any traffic**.
- Gateway VPC endpoints (S3, DynamoDB): **$0.00**—no hourly, no data charge.
- Interface VPC endpoints (PrivateLink): **$0.01/hour per AZ + $0.01/GB** (first 1 PB tier).
- Break-even per Interface endpoint at 2 AZs: ~417 GB/mo, but the $0.035/GB saving vs NAT begins immediately—high-volume services (ECR, Logs) win fast.
- Flow Logs to S3: per-GB delivery plus storage; records are metadata, so 600s aggregation and a 90d lifecycle keep a typical startup under ~$1/mo.

## Troubleshooting

- **Athena query returns zero rows.** Cause: default flow-log format lacks `pkt-dstaddr`/`flow-direction`. Fix: recreate the flow log with the generated custom format, wait one interval, re-query.
- **SDK calls still go through NAT after creating an Interface endpoint.** Cause: `private_dns_enabled` is false. Fix: set it true; ensure VPC `enableDnsSupport`/`enableDnsHostnames` are on.
- **S3/DynamoDB traffic unchanged after adding the Gateway endpoint.** Fix: add every private route table to the endpoint's `route_table_ids`.
- **Interface endpoint creation fails: service not available in region.** Fix: verify with `describe-vpc-endpoint-services`; keep NAT for that single service.
- **NAT bill didn't drop after apply.** Cause: a non-AWS destination (third-party API, package registry) is the real driver—shown as an unresolved EXTERNAL range. Fix: consolidate onto one shared NAT or add a partner PrivateLink endpoint.
