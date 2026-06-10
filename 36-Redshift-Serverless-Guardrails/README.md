# Redshift Serverless Cost Guardrails: RPU Caps, Daily Usage Limits, and True Scale-to-Zero

Three isolated Redshift Serverless workgroups with hard daily RPU-hour caps, right-sized base/max compute, and a warehouse that genuinely sits at $0 when idle—analytics that bills for seconds of work, never for idle hours.

## The Problem

Redshift Serverless is marketed as "pay only for what you use," but the console default workgroup ships with a **base capacity of 128 RPUs**. At $0.375 per RPU-hour in us-east-1, that is **$48.00 per hour** every time a query runs—and Serverless bills a 60-second minimum each time compute resumes. A BI tool auto-refreshing dashboards 10 hours a day against that default burns roughly **$480/day (~$14,400/month)** before anyone notices, because Serverless has no pause button, no instance to stop, and no native "off" switch. The guardrails exist—`max-capacity`, per-workgroup usage limits with a `deactivate` breach action, workload isolation via data sharing—but they are spread across three APIs and almost nobody wires all of them. This prompt generates the complete, production-ready Terraform stack that makes overspending structurally impossible: every workload gets its own workgroup, its own RPU ceiling, and its own daily dollar cap that hard-stops queries at breach.

## Who This Is For

- Startups adopting Redshift Serverless who need the bill bounded **before** finance asks
- Data engineers replacing an always-on provisioned cluster and expecting scale-to-zero to "just work"
- Platform teams who must give analysts ad hoc SQL access without handing them an uncapped warehouse

## How to Use

1. **Kiro CLI**: save the System Prompt below as `redshift-guardrails.md` in your project, then run `kiro chat` and reference it: "Implement redshift-guardrails.md against my account."
2. **Claude Code / Cursor / any AI assistant**: paste the System Prompt directly into a session opened in your Terraform repo. The prompt instructs the assistant to inspect your account first, so run it where AWS credentials are configured.
3. Edit the six values that matter before sending: region, monthly budget, the three base/max RPU pairs, and the three daily RPU-hour caps.
4. Review the generated `terraform plan` output—confirm 2 namespaces minimum, 3 workgroups, and **6 usage limits** appear before applying.
5. Run the generated `verify-guardrails.sh` and keep the PASS output as your audit evidence.

**Prerequisites**

- **Required Access**: IAM permissions for `redshift-serverless:*`, `budgets:*`, `cloudwatch:PutMetricAlarm`, `sns:*`, `kms:CreateKey`, `secretsmanager:GetSecretValue`, and `ec2:Describe*` (subnet/security-group lookups)
- **Recommended Background**: basic Terraform, SQL, and a one-paragraph understanding of RPUs (Redshift Processing Units—the Serverless compute/billing unit)
- **Tools Required**: Terraform >= 1.5 with AWS provider >= 5.0, AWS CLI v2, and the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) so the assistant verifies current RPU pricing and API shapes instead of guessing

**Key Parameters:** Region (us-east-1), monthly budget ($600 w/ 80%/100% alerts), base/max RPUs per workgroup (BI 8/32, ETL 32/64, ad hoc 8/16), daily RPU-hour caps (24/48/12, breach_action=deactivate), monthly trend limits (breach_action=emit-metric), port (5439), data size (2 TB).

**Troubleshooting:** If the assistant generates two workgroups attached to one namespace, that is a real API constraint violation—Redshift Serverless binds exactly one workgroup per namespace. The data-sharing topology in the prompt (producer namespace + consumer workgroups) is the correct fix, not a workaround.

## System Prompt

```
# Amazon Redshift Serverless Cost-Guardrail Architecture Request

## Project Overview
Design production-ready Amazon Redshift Serverless infrastructure for a startup analytics platform: ~2 TB of data and three workloads that must never contend for compute or share a bill line — BI dashboards (bursty, business hours), hourly ELT loads, and ad hoc data science. Compute must cost $0 when idle, and each workload must be physically incapable of exceeding its daily spend ceiling. Region: us-east-1. IaC: Terraform, AWS provider >= 5.0.

## Verify Reality Before Generating
1. Run `aws sts get-caller-identity` and `aws redshift-serverless list-workgroups` to confirm credentials, region, and existing resources. If a `default` workgroup exists at the console-default 128 RPU base capacity, flag it FIRST: at $0.375/RPU-hour that is $48.00/hour whenever a query runs.
2. Confirm Redshift Serverless availability and the current RPU price for my region via the AWS Documentation MCP server (aws_read_documentation) — never quote prices from memory.
3. Confirm the AWS provider version in my lockfile supports aws_redshiftserverless_usage_limit before emitting Terraform. If any check cannot be completed, state what failed and stop — do not invent resource names or attributes.

## Detailed Requirements
### 1. Workload Isolation Topology
- One producer namespace `analytics-core` owns all tables. Consumer workgroups get read access through Redshift data sharing (CREATE DATASHARE / GRANT USAGE ... TO NAMESPACE). A namespace binds to exactly one workgroup — never attach two workgroups to one namespace.
- Three workgroups: `wg-bi-serve`, `wg-etl-load`, `wg-adhoc-lab`. All in private subnets across 3 AZs, `publicly_accessible = false`, one dedicated security group each allowing port 5439 only from the application/VPN CIDR.

### 2. Base/Max RPU Sizing
- `wg-bi-serve`: base_capacity 8, max_capacity 32. `wg-etl-load`: base 32, max 64. `wg-adhoc-lab`: base 8, max 16.
- Base is the floor compute resumes at; max_capacity is the hard autoscale ceiling. Valid base range is 4 RPUs, then in units of 8 from 8 up to 512 RPUs (up to 1024 in us-east-1 and a few other Regions). Justify each pair in one line and emit them as Terraform variables with these defaults.

### 3. Per-Workgroup Cost Caps (hard stop)
- For each workgroup, create `aws_redshiftserverless_usage_limit` with usage_type `serverless-compute`, period `daily`, breach_action `deactivate`: 24 RPU-hours (BI), 48 (ETL), 12 (ad hoc). Worst-case compute is therefore capped at 84 RPU-hours/day = $31.50/day.
- Add a second monthly limit per workgroup with breach_action `emit-metric` (6 limits total) so trends alarm without silently killing ETL mid-month.

### 4. Pause-Equivalent Behavior
- Compute must return to $0 within minutes of the last query: no scheduled queries, no materialized views with AUTO REFRESH YES on the BI path, and health checks must read CloudWatch metrics — a `SELECT 1` keep-alive every 60 seconds re-triggers the 60-second minimum charge indefinitely and defeats scale-to-zero.
- Document the only always-on cost: Redshift managed storage at $0.024/GB-month (~$49/month for 2 TB). Open JDBC connections alone do not bill; only active query processing does.

### 5. Monitoring & Budgets
- AWS Budget `redshift-serverless-monthly` at $600 with 80% and 100% actual-spend alerts to an SNS topic with email subscription.
- CloudWatch alarms per workgroup in the AWS/Redshift-Serverless namespace: ComputeCapacity pinned at max_capacity for 15 minutes, and daily ComputeSeconds anomaly.

### 6. Security Baseline
- Encrypt the namespaces with a customer-managed KMS key. Use the managed admin password option (credentials in AWS Secrets Manager) — never hardcode passwords. Attach one IAM role for COPY, scoped to a single S3 landing bucket.

## Error Management
- On any AWS API or Terraform error, print the exact failing command and the error verbatim, then propose the fix — never retry blindly.
- If usage-limit creation fails with ValidationException, check the workgroup has reached AVAILABLE status and add depends_on rather than removing the limit.

## Deliverables Requested
1. `redshift-serverless.tf`, `usage-limits.tf`, `monitoring.tf`, `budgets.tf`, `variables.tf` (defaults = values above)
2. `verify-guardrails.sh` — AWS CLI checks printing PASS/FAIL for every cap, limit, and alarm
3. `cost-model.md` — expected vs worst-case monthly cost table at the verified RPU price
4. `breach-runbook.md` — exactly what users see when `deactivate` fires, and the two commands (update-usage-limit / delete-usage-limit) that restore service deliberately

## Acceptance Evidence
Close with the verification commands and their expected output: `list-usage-limits` returning 6 limits, `get-workgroup` showing each base/max pair, and the budget alert confirmed. The full stack must `terraform apply` cleanly in under 15 minutes. Output the files directly without any preamble.
```

## What You Get

1. **`redshift-serverless.tf`** — producer + consumer namespaces, 3 workgroups with explicit `base_capacity`/`max_capacity`, private networking, KMS encryption
2. **`usage-limits.tf`** — 6 `aws_redshiftserverless_usage_limit` resources: 3 daily `deactivate` hard caps + 3 monthly `emit-metric` trend limits
3. **`budgets.tf`** — $600 monthly AWS Budget with 80%/100% SNS email alerts scoped to Redshift spend
4. **`monitoring.tf`** — per-workgroup CloudWatch alarms on ComputeCapacity saturation and ComputeSeconds
5. **`variables.tf`** — every tunable (RPU pairs, caps, budget, CIDR) as a variable with production defaults
6. **`verify-guardrails.sh`** — one-command PASS/FAIL audit of every guardrail
7. **`cost-model.md`** — expected vs worst-case monthly spend table
8. **`breach-runbook.md`** — the 2 AM page answered: what `deactivate` looks like and how to lift it on purpose

## Example Output

```
$ ./verify-guardrails.sh
[PASS] wg-bi-serve    base=8  max=32   daily cap=24 RPU-h (deactivate)
[PASS] wg-etl-load    base=32 max=64   daily cap=48 RPU-h (deactivate)
[PASS] wg-adhoc-lab   base=8  max=16   daily cap=12 RPU-h (deactivate)
[PASS] 6/6 usage limits present (3 daily/deactivate, 3 monthly/emit-metric)
[PASS] Budget redshift-serverless-monthly: $600, alerts at 80%/100%
[PASS] No workgroup publicly accessible; all namespaces KMS-encrypted
Worst-case compute ceiling: 84 RPU-h/day = $31.50/day ($945/mo). Idle compute: $0.00.
```

## AWS Services Used

Amazon Redshift Serverless, AWS Budgets, Amazon CloudWatch, Amazon SNS, AWS Key Management Service (KMS), AWS Secrets Manager, AWS Identity and Access Management (IAM), Amazon VPC, Amazon S3

## Well-Architected Alignment

- **Cost Optimization** — base/max RPU right-sizing per workload, daily `deactivate` usage limits as a hard spend ceiling, scale-to-zero verified rather than assumed, budget alerts at 80%/100%
- **Operational Excellence** — `verify-guardrails.sh` makes guardrail state auditable in one command; `breach-runbook.md` turns the breach event into a documented procedure
- **Security** — KMS-encrypted namespaces, Secrets Manager-managed admin credentials, no public endpoints, security groups restricted to port 5439 from known CIDRs, single-bucket COPY role
- **Reliability** — workload isolation via data sharing means a runaway ad hoc query can exhaust only its own workgroup's cap, never the ETL pipeline's compute
- **Performance Efficiency** — each workload scales independently to its own max_capacity instead of contending on one oversized cluster

## Cost Notes

All figures us-east-1, verified June 2026: **$0.375 per RPU-hour**, billed per second with a **60-second minimum** each time compute resumes; managed storage **$0.024/GB-month**.

- **Idle**: compute $0.00; 2 TB storage ≈ **$49/month** ($1.64/day) — that is the entire weekend bill
- **Typical**: BI ~16 RPU-h/day ($6.00) + ETL ~24 RPU-h/day ($9.00) + ad hoc ~4 RPU-h/day ($1.50) ≈ **$495/month compute, ~$545 total** — inside the $600 budget
- **Worst case (capped)**: 84 RPU-h/day = $31.50/day = **$945/month** — the absolute ceiling the `deactivate` limits enforce, vs $14,400/month for an unguarded default workgroup
- Alert-only AWS Budgets are free; the CloudWatch alarms add ~$0.10/alarm/month

## Troubleshooting

1. **Compute never reaches $0** — Cause: a `SELECT 1` health check or AUTO REFRESH materialized view resumes compute repeatedly, each resume billing the 60-second minimum. Fix: delete keep-alive pings, run `ALTER MATERIALIZED VIEW ... AUTO REFRESH NO` on the BI consumer, and monitor health from CloudWatch ComputeSeconds instead.
2. **`terraform apply` fails with ValidationException on a usage limit** — Cause: the limit was created before its workgroup reached AVAILABLE. Fix: add `depends_on` to the workgroup resource and re-apply; the limit attaches cleanly on the second pass.
3. **Queries suddenly rejected mid-afternoon** — Cause: the daily `deactivate` cap breached — the guardrail working as designed. Fix: follow `breach-runbook.md`: confirm the spend is legitimate, then `aws redshift-serverless update-usage-limit --amount <higher>` or wait for the daily reset.
4. **Creating a second workgroup on the same namespace fails** — Cause: Redshift Serverless enforces a 1:1 namespace-to-workgroup binding. Fix: give each workload its own namespace and share tables from `analytics-core` with `CREATE DATASHARE` + `GRANT USAGE ... TO NAMESPACE`.
5. **ETL slower than the old provisioned cluster** — Cause: max_capacity 64 is the autoscale ceiling and the load wants more. Fix: raise `max_capacity` in one variable, re-apply, and let the ComputeCapacity saturation alarm tell you when you have headroom again.
