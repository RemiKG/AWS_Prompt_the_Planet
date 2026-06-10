# The $0 Validation Stack: Lambda, DynamoDB, and CloudFront Inside the AWS Free Plan With Alarms That Fire Before the First Cent

A production-ready serverless MVP engineered to fit AWS always-free limits under the current account model, with three AWS Budgets alarms that fire before money moves—so you validate for six months on a cash bill of exactly $0.00.

## The Problem

On July 15, 2025, AWS retired the legacy 12-month free tier for new accounts. New accounts now pick a **Free plan** or a **Paid plan**: the Free plan grants **$100 in credits at sign-up**, earnable to **$200** via five $20 activities (Amazon EC2, Amazon RDS, AWS Lambda, Amazon Bedrock, AWS Budgets), and the account **auto-closes at 6 months or when credits run out—whichever comes first**. Most tutorials and AI assistants still describe the pre-2025 model. Two failure modes kill the runway:

1. **Silent credit burn.** A DynamoDB table left in its on-demand default (not covered by always-free), a log group on "Never expire" retention, or one forgotten NAT Gateway at $0.045/hr (~$32.40/month) eats the entire $100 credit in ~3 months with zero users.
2. **The netted-$0 illusion.** Cost Explorer shows $0.00 because credits net out gross usage—you discover the real burn rate the day the credits die and the Free plan account closes around your data.

This prompt makes the AI assistant architect against the **current** rules: every resource either fits an always-free limit or is flagged with its exact credit-burn rate, and three budget alarms fire before the first cent of real spend.

## Who This Is For

- Pre-seed founders and indie hackers validating an idea before spending runway
- Students and side-project developers on accounts created after July 15, 2025
- Anyone who has opened a surprise AWS bill and wants the alarm to come first this time

## How to Use

1. Open Kiro CLI, Claude Code, or Amazon Q Developer CLI in an empty directory (or save the prompt as `.kiro/steering/zero-bill-mvp.md`).
2. Paste the System Prompt below; replace `[APP_NAME]`, `[EXPECTED_MONTHLY_USERS]`, and `[LAUNCH_REGION]`.
3. Let the assistant run its reality checks (account model, credentials, runtime) **before** it writes any files.
4. Review the ledger and cost map, then deploy: `sam build && sam deploy --guided`, then `bash budgets.sh`, then `bash verify.sh`.
5. Click the SNS confirmation link within 3 days, or the alarms can never deliver.

**Prerequisites**

- **Required Access:** an IAM principal with CloudFormation, Lambda, DynamoDB, S3, CloudFront, API Gateway, SNS, `budgets:*`, and `freetier:GetFreeTierUsage`; "IAM access to billing" activated in account settings.
- **Recommended Background:** basic serverless concepts; ~30 minutes end to end.
- **Tools Required:** AWS CLI v2, AWS SAM CLI (1.100+), Node.js 22; the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) so the assistant verifies current free-plan limits instead of reciting stale ones.

**Key Parameters:** Lambda memory (128 MB) and timeout (10 s), DynamoDB provisioned throughput (5 RCU/5 WCU; 25/25 always-free account ceiling), log retention (7 d), CloudFront price class (PriceClass_100), budget thresholds ($0.01 actual / $5 forecast / $20 gross with Credits unchecked), Free plan window (6 months).

**Troubleshooting:** If the assistant generates the table with `BillingMode: PAY_PER_REQUEST`, that is the common template default—on-demand is NOT covered by always-free, which applies only to 25 RCU/25 WCU of **provisioned** capacity. Re-prompt: "BillingMode must be PROVISIONED at 5/5, autoscaling disabled."

## System Prompt

```
# Zero-Bill MVP Architecture Design Request

## Project Overview
Act as a senior AWS solutions architect. Design and generate a production-ready
serverless MVP for [APP_NAME] that runs at exactly $0.00/month cash cost during
validation by fitting inside AWS always-free limits under the CURRENT account
model: accounts created on or after July 15, 2025 choose a Free plan or a Paid
plan; the Free plan grants $100 in credits at sign-up (earnable to $200 via five
$20 activities, including creating an AWS Budget) and auto-closes at 6 months or
credit exhaustion, whichever comes first. The legacy 12-month free tier does NOT
apply to new accounts. Stack: static frontend on S3 behind CloudFront, JSON API
on API Gateway HTTP API + Lambda, data in one DynamoDB table. Expected scale:
[EXPECTED_MONTHLY_USERS] monthly users in [LAUNCH_REGION] (default us-east-1).

## Verify Reality First — before generating a single file
1. Identify my account model: Free plan, Paid plan, or legacy 12-month account
   (created before July 15, 2025). Run `aws freetier get-free-tier-usage`; if it
   errors, ask me to read Billing and Cost Management > Free Tier and report
   back. Never assume the pre-2025 model.
2. Run `aws sts get-caller-identity` and `sam --version` to confirm working
   credentials and SAM CLI 1.100+.
3. Confirm the nodejs22.x Lambda runtime is available in the target region using
   the AWS Documentation MCP tools (aws_read_documentation). If any limit or
   price below may have changed, verify it there first. Never invent a free-tier
   number; if unverifiable, say so.
4. If any check fails: stop, report exactly what failed and why, and wait.

## Detailed Requirements
### 1. Always-Free Core (hard constraints)
- Lambda: nodejs22.x, arm64, 128 MB, 10 s timeout — always free to 1M requests
  + 400,000 GB-seconds/month (at 128 MB that is 3.2M compute-seconds; the 1M
  request cap binds first)
- DynamoDB: BillingMode PROVISIONED at 5 RCU / 5 WCU, autoscaling disabled —
  always free covers 25 RCU / 25 WCU / 25 GB account-wide; on-demand
  (PAY_PER_REQUEST) is NOT covered and is the #1 silent credit burner
- CloudFront: PriceClass_100, Origin Access Control to a private S3 bucket —
  always free to 1 TB egress + 10M requests/month
- CloudWatch Logs: explicit RetentionInDays: 7 on every log group (the default
  "Never expire" walks past the 5 GB free ingestion)
### 2. Credit Consumers (allowed, but flagged)
- API Gateway HTTP API ($1.00 per million requests) and S3 ($0.023/GB-month
  plus request fees) consume credits, not always-free. List both in the ledger
  with exact rates, and document a Lambda Function URL (no per-request charge)
  as the $0 fallback path.
### 3. Alarms That Fire Before Money Moves
- Budget 1: the "Zero spend budget" template — alert at $0.01 of actual spend
- Budget 2: $5 monthly cost budget with a FORECASTED-threshold alert at 100% —
  it fires before the spend exists
- Budget 3: $20 monthly budget with Credits UNCHECKED in advanced options — it
  tracks gross usage (my credit burn rate), because credit-netted cost reads
  $0.00 right up until the credits die
- Route all three to my email through one SNS topic. Remind me that completing
  the AWS Budgets activity itself earns $20 of Free plan credit.
### 4. Security
- No public S3 bucket. OAC bucket policy for principal cloudfront.amazonaws.com
  restricted with an AWS:SourceArn condition on the distribution ARN. One
  least-privilege Lambda execution role allowing only dynamodb:GetItem, PutItem,
  and Query on the single table ARN.

## Deliverables Requested
1. template.yaml — one complete AWS SAM template, deployable with
   `sam deploy --guided`
2. FREE-PLAN-LEDGER.md — per-service table: always-free limit vs
   credit-consuming rate, my projected usage, headroom %
3. COST-MAP.md — ranked "first thing that costs money" map: Route 53 hosted
   zone $0.50/mo, NAT Gateway $0.045/hr (~$32/mo), public IPv4 $0.005/hr
   (~$3.65/mo), Secrets Manager $0.40/secret/mo, KMS customer key $1/mo,
   DynamoDB on-demand mode, CloudWatch Logs past 5 GB at $0.50/GB ingested
4. UPGRADE-TRIGGERS.md — honest thresholds, each with the exact next step and
   its first real monthly cost: sustained >20 writes/sec (80% of the 25 WCU
   ceiling), >800K API requests/month, >800 GB CloudFront egress/month, custom
   domain needed, and month 5 of the 6-month Free plan window (convert to the
   Paid plan deliberately, before auto-close decides for you)
5. budgets.sh — AWS CLI commands creating all three budgets plus the SNS topic
   and email subscription
6. verify.sh — acceptance evidence: curl the CloudFront URL expecting HTTP 200,
   describe-table proving PROVISIONED 5/5, describe-budgets proving 3 budgets,
   get-free-tier-usage summary, and a final printed line:
   "PROJECTED CASH BILL: $0.00/month"

## Error Management
On any deploy failure, output Cause -> Resolution and stop; no blind retries.
If a resource cannot stay inside always-free limits at my stated scale, say so
plainly and show the real monthly number — honesty over a fake $0.

The full stack must deploy and verify in under 15 minutes. Output the files
directly, without preamble.
```

## What You Get

1. **template.yaml** — complete AWS SAM template: Lambda (nodejs22.x, arm64, 128 MB), HTTP API, DynamoDB pinned to PROVISIONED 5/5, private S3 bucket, CloudFront with OAC, 7-day log retention
2. **FREE-PLAN-LEDGER.md** — the always-free vs credit-consuming ledger, per service, with actual limits and projected headroom
3. **COST-MAP.md** — the ranked first-thing-that-costs-money map with exact rates
4. **UPGRADE-TRIGGERS.md** — honest thresholds for when $0 stops being the right answer, each with its first real monthly cost
5. **budgets.sh** — three budgets (zero-spend actual, $5 forecast, $20 gross-usage) plus SNS topic and subscription, as CLI commands
6. **verify.sh** — acceptance evidence: HTTP 200 from CloudFront, PROVISIONED 5/5 from `describe-table`, 3 budgets from `describe-budgets`, a Free Tier usage summary
7. **README.md** — deploy order, the 6-month Free plan calendar, and the convert-to-Paid decision checklist

## Example Output

```
FREE-PLAN-LEDGER (us-east-1, account model: Free plan, month 2 of 6)
Service        Status            Limit                      Projected      Headroom
Lambda         ALWAYS FREE       1M req + 400K GB-s/mo      90K req        91%
DynamoDB       ALWAYS FREE       25 RCU/25 WCU/25 GB        5/5, 0.4 GB    80%
CloudFront     ALWAYS FREE       1 TB egress + 10M req/mo   18 GB          98%
API Gateway    CREDIT-CONSUMING  $1.00/M requests           90K = $0.09/mo
S3             CREDIT-CONSUMING  $0.023/GB-mo + requests    40 MB < $0.01/mo

PROJECTED CASH BILL: $0.00/month   CREDIT BURN: ~$0.10/month
verify.sh: CloudFront 200 OK | Table PROVISIONED 5/5 | Budgets active: 3
```

## AWS Services Used

AWS Lambda, Amazon DynamoDB, Amazon S3, Amazon CloudFront, Amazon API Gateway, AWS Budgets, Amazon CloudWatch, Amazon SNS, AWS IAM, AWS CloudFormation (SAM)

## Well-Architected Alignment

- **Cost Optimization:** every resource mapped to an always-free limit or an exact credit rate; forecast alarms precede spend; upgrade triggers prevent both overpaying and false economy
- **Operational Excellence:** budgets-as-code, a scripted verification gate (`verify.sh`), and a reality-check phase before any generation
- **Security:** private S3 with Origin Access Control and an `AWS:SourceArn` condition; one least-privilege Lambda role scoped to three actions on one table ARN
- **Reliability:** fully managed serverless services; the 5 WCU provisioned cap doubles as deliberate backpressure during validation
- **Performance Efficiency:** CloudFront edge caching and arm64 Lambda price-performance
- **Sustainability:** scale-to-zero compute—idle cost and idle carbon are both zero

## Cost Notes

- **During validation:** cash bill $0.00/month; credit burn ~$0.10/month at 90K API requests ($0.09 API Gateway + under $0.01 S3)—the $100 sign-up credit outlasts the 6-month window ~1,000x at that rate.
- **First cash dollar candidates:** Route 53 hosted zone $0.50/month, Secrets Manager $0.40/secret/month, KMS customer-managed key $1.00/month.
- **Runway killers this prevents:** NAT Gateway ~$32.40/month idle, one public IPv4 $3.65/month, DynamoDB on-demand at ~$0.625 per million writes when provisioned would have been free.
- **At 10x scale (1M API requests, 200 GB egress):** roughly $1.00/month gross—API Gateway $1.00, everything else still always-free.

## Troubleshooting

- **Budget alert emails never arrive.** Cause: SNS subscription unconfirmed (the link expires after 3 days), or you expected real time—billing data refreshes up to three times daily. Fix: re-subscribe, click the link, allow hours of latency.
- **Charges appear despite "always free."** Cause: table in on-demand mode, or a "Never expire" log group crossed the 5 GB free ingestion; the 25 RCU/WCU allowance is account-wide. Fix: `BillingMode: PROVISIONED` at 5/5, 7-day retention, one region.
- **CloudFront returns AccessDenied XML.** Cause: bucket policy missing or its `AWS:SourceArn` does not match the distribution ARN. Fix: apply the generated policy for principal `cloudfront.amazonaws.com` with the exact ARN, then invalidate `/*`.
- **`ProvisionedThroughputExceededException` under load.** Cause: the 5 WCU cap plus exhausted burst—the guardrail working. Fix: sustained >~20 writes/sec is the documented upgrade trigger; switch to on-demand knowingly at ~$0.625 per million writes.
- **`aws freetier get-free-tier-usage` returns AccessDenied.** Cause: missing `freetier:GetFreeTierUsage` or IAM billing access not activated. Fix: attach the permission and enable "IAM access to billing" in account settings, or read Billing console > Free Tier.

---

**Bottom line:** the alarm fires before the bill exists. Validate the idea; keep the runway.
