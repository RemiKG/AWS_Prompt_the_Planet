# Cost-Capped Spark ETL on EMR Serverless: Right-Sized Workers, 15-Minute Auto-Stop, No Idle-Cluster Bill

Right-sizes EMR Serverless workers per job profile, keeps the 15-minute auto-stop, hard-caps concurrent vCPUs, and prices every run to the cent—so heavy Spark ETL costs $1.44 a night instead of $2,875 a month.

## The Problem

Teams that outgrow lighter ETL services move heavy Spark work to EMR Serverless for full engine control—custom Spark configs, big shuffles, 120 GB workers—then make one of three expensive mistakes. They accept the default executor size (4 vCPU / 14 GB) and OOM on a skewed join at 2 AM. They turn on pre-initialized capacity "for fast starts" and discover a warm pool of 1 driver + 2 executors bills $0.91/hour even when zero jobs run—$663/month for nothing; a full 13-worker fleet held warm is $2,875/month. Or they submit their first real job into a fresh account and it sits in SCHEDULED forever because the "Max concurrent vCPUs per account" Service Quota often starts at just 16 vCPUs, below a single mid-size job's footprint.

EMR Serverless bills per second per worker ($0.052624/vCPU-hour + $0.0057785/GB-hour in us-east-1), which means right-sizing IS the cost model. This prompt makes an AI assistant do the sizing math, the pre-initialized-vs-cold-start decision, and the cost cap before anything is deployed—and forces it to reconcile the predicted cost against the actual `totalResourceUtilization` the API reports after the first run.

## Who This Is For

- Data engineers running Spark jobs too heavy or too custom for managed ETL defaults: multi-hundred-GB shuffles, pinned Spark versions, custom JARs
- Startups with nightly/hourly batch pipelines who need the bill to track runs, not wall-clock time
- Platform engineers told to "cap what the data team can spend" without blocking their jobs

## How to Use

1. Paste the System Prompt below into Kiro CLI, Claude Code, or any AI assistant with AWS CLI v2 access and the AWS Documentation MCP server configured.
2. Replace the bracketed placeholders: [RAW_BUCKET], [CURATED_BUCKET], [REGION], [ALERT_EMAIL]. If your job profile differs from the stated 150 GB gzip-JSON job, edit the Project Overview paragraph—the sizing rules adapt to what you state.
3. Let the assistant run the Verify Reality First checks (release label, vCPU quota, bucket existence) before accepting any generated files. If it skips them, tell it to start over at the gate.
4. Review the six artifacts, run `create_application.sh` (or `terraform apply`), then `submit_job.sh`.
5. After the first SUCCESS, run the VERIFY.md checklist and reconcile `get-job-run` resource utilization against COST_MODEL.md.

**Prerequisites**

- **Required Access:** IAM permissions for `emr-serverless:*`, `iam:CreateRole` / `iam:PassRole` on the job execution role, `servicequotas:ListServiceQuotas`, `budgets:CreateBudget`, and read/write on the two S3 buckets
- **Recommended Background:** basic PySpark (read/repartition/write), S3 prefix layout for partitioned data
- **Tools Required:** AWS CLI v2, AWS Documentation MCP server (aws_read_documentation, aws_search_documentation); Terraform >= 1.5 if you choose the Terraform output path

**Key Parameters:** Auto-stop idle timeout (15 min default, range 1–10080, never disable), worker size (4 vCPU / 16 GB default; 1–16 vCPU, 2–120 GB per worker), per-worker disk (20 GB free, `spark.emr-serverless.executor.disk` up to 200 GB), `maximumCapacity` (60 vCPU / 240 GB / 600 GB), `executionTimeoutMinutes` (90), `spark.dynamicAllocation.maxExecutors` (12), `spark.sql.shuffle.partitions` (400), pre-init warm pool (1 driver + 2 executors, $0.91/hr), monthly budget ($75 with 80%/100% alerts).

**Troubleshooting:** If your first job sits in SCHEDULED for several minutes on a fresh account, that's expected—the "Max concurrent vCPUs per account" Service Quota frequently starts at 16, below this job's 52-vCPU peak. The prompt checks the quota first and emits the Service Quotas increase request (400 vCPUs) before generating anything else.

## System Prompt

```
# EMR Serverless Spark ETL — Architecture & Build Request

## Project Overview
Generate a production-ready Spark ETL deployment on Amazon EMR Serverless for a nightly batch job: convert 150 GB of gzip-compressed JSON under s3://[RAW_BUCKET]/events/ into snappy Parquet partitioned by event_date under s3://[CURATED_BUCKET]/events/, target wall time under 25 minutes. The deployment must be right-sized, auto-stopped, and cost-capped — never an always-on cluster. Region: [REGION]. Use Terraform for infrastructure and plain AWS CLI v2 for job submission.

## Verify Reality First (hard gate — before generating anything)
1. Resolve the latest EMR 7.x release label and its Spark version with the AWS Documentation MCP server (aws_read_documentation against the EMR Serverless release notes). Treat emr-7.2.0 (Spark 3.5.1) as the floor — never guess version strings.
2. Check the "Max concurrent vCPUs per account" quota: `aws service-quotas list-service-quotas --service-code emr-serverless`. If it is below 60, emit the Service Quotas increase request (400 vCPUs) before anything else.
3. Confirm both buckets exist (`aws s3api head-bucket --bucket [RAW_BUCKET]`) and that no application named nightly-events-etl already exists (`aws emr-serverless list-applications`).
If any check fails, stop, print the failing command and the exact remediation. Do not generate files against unverified assumptions.

## Detailed Requirements

### 1. Application
- `--type SPARK`, `--architecture X86_64`, auto-start enabled, `--auto-stop-configuration '{"enabled": true, "idleTimeoutMinutes": 15}'`. Keep the 15-minute default; never disable auto-stop.
- Hard concurrency ceiling: `--maximum-capacity '{"cpu": "60 vCPU", "memory": "240 GB", "disk": "600 GB"}'`.

### 2. Worker Sizing by Job Profile
- This job (parse-heavy, narrow transforms): 1 driver + 12 executors at 4 vCPU / 16 GB each; `spark.executor.memory=14g` (the default 10% memory overhead must fit inside the 16 GB worker); default 20 GB disk per worker (free).
- Shuffle/join-heavy profile: 4 vCPU / 30 GB. Skewed wide aggregations: 8 vCPU / 60 GB. Absolute per-worker maximum: 16 vCPU / 120 GB.
- Set `spark.dynamicAllocation.maxExecutors=12` and `spark.sql.shuffle.partitions=400`. Repartition immediately after read — gzip is non-splittable, so input-stage parallelism equals file count.

### 3. Pre-Initialized Capacity Decision
State which side this job lands on and show the math: a warm pool of 1 driver + 2 executors (12 vCPU / 48 GB) bills $0.91/hour while idle and removes the typical 60–120 second cold start. Use pre-initialized capacity only when jobs arrive more than 4x/hour or an SLA requires sub-30-second starts. A once-nightly job takes the cold start and pays $0.00 idle.

### 4. Cost Controls
- `--execution-timeout-minutes 90` on every start-job-run so a wedged job cannot burn all night.
- AWS Budgets: $75/month with 80% and 100% alerts to an SNS topic subscribed by [ALERT_EMAIL].
- Logs: s3MonitoringConfiguration to s3://[CURATED_BUCKET]/emr-logs/ plus free managed log persistence (30-day retention).

### 5. Security
Execution role trusted only by emr-serverless.amazonaws.com; s3:GetObject on the raw prefix, s3:PutObject on the curated and log prefixes only — no AmazonS3FullAccess, no bucket wildcards.

## Cost Math (required deliverable, line by line, us-east-1 rates)
At $0.052624 per vCPU-hour and $0.0057785 per GB-hour with the first 20 GB of disk per worker free: 13 workers x 4 vCPU x 22 min = 19.07 vCPU-hours = $1.00; 13 workers x 16 GB x 22 min = 76.27 GB-hours = $0.44; total ~$1.44 per run, ~$43/month at nightly cadence. Contrast with the avoided failure: the same fleet held warm 24/7 = $3.94/hour = ~$2,875/month.

## Deliverables
1. main.tf — aws_emrserverless_application plus the execution role and trust policy
2. iam_policy.json — least-privilege execution role policy document
3. etl_job.py — PySpark skeleton: read gzip JSON, repartition(400), write snappy Parquet partitioned by event_date
4. submit_job.sh — start-job-run with sparkSubmitParameters, execution timeout, and log configuration
5. COST_MODEL.md — the per-run math above plus a warm-pool break-even table
6. VERIFY.md — the acceptance checklist below plus an Error Management table

## Error Management
In VERIFY.md, cover each failure mode as Cause -> Resolution: job stuck in SCHEDULED (account vCPU quota or maximumCapacity too low), executor OOM (memory-per-core undersized for the shuffle profile), "No space left on device" (20 GB default disk ceiling), and S3 AccessDeniedException (execution role prefix scoping).

## Acceptance Evidence
Prove it works: `aws emr-serverless get-application` shows autoStopConfiguration enabled with idleTimeoutMinutes 15; `aws emr-serverless get-job-run` shows state SUCCESS and a totalResourceUtilization block — reconcile vCPUHour x $0.052624 + memoryGBHour x $0.0057785 against the predicted $1.44 and flag drift over 20%; `aws s3 ls` shows event_date= partitions in the curated bucket; `get-application` 20 minutes after job end shows state STOPPED. A fresh account must go from zero to a submitted job in under 15 minutes. Output the files only — no preamble.
```

## What You Get

1. **main.tf** — `aws_emrserverless_application` with auto-stop (15 min), auto-start, the 60-vCPU/240-GB maximum-capacity ceiling, and the execution role with least-privilege S3 policy
2. **iam_policy.json** — standalone policy document scoped to the raw, curated, and log prefixes
3. **etl_job.py** — PySpark skeleton handling the non-splittable-gzip repartition correctly
4. **submit_job.sh** — `aws emr-serverless start-job-run` with all Spark conf flags, the 90-minute execution timeout, and S3 log configuration
5. **COST_MODEL.md** — per-run cost computed line by line, plus the pre-initialized-vs-cold-start break-even table
6. **VERIFY.md** — acceptance checklist with the exact CLI commands and expected outputs, plus a Cause → Resolution error table

## Example Output

```
COST PER RUN — nightly-events-etl (us-east-1, X86_64)
Compute: 13 workers x 4 vCPU x 0.367 hr = 19.07 vCPU-h x $0.052624 = $1.00
Memory:  13 workers x 16 GB  x 0.367 hr = 76.27 GB-h  x $0.0057785 = $0.44
Storage: 20 GB/worker (within free tier)                           = $0.00
TOTAL: $1.44/run -> $43.32/month at nightly cadence
Avoided: same fleet pre-initialized 24/7 = $3.94/hr = $2,875/month
Decision: COLD START. Job runs 1x/day; the ~90s startup penalty is
free, while a minimal warm pool would cost $663/month idle.
```

## AWS Services Used

Amazon EMR Serverless, Amazon S3, AWS Identity and Access Management (IAM), Amazon CloudWatch, AWS Budgets, Amazon SNS, AWS Service Quotas

## Well-Architected Alignment

- **Cost Optimization:** per-second billing exploited via 15-minute auto-stop, hard `maximumCapacity` ceiling, $75 budget with 80%/100% alerts, and post-run reconciliation of predicted cost against the API's `totalResourceUtilization`
- **Performance Efficiency:** worker vCPU/memory sized per job profile (2 GB/core parse work up to 7.5 GB/core skewed aggregations), gzip non-splittability handled with an explicit repartition, dynamic allocation capped at 12 executors
- **Operational Excellence:** idempotent IaC, S3 plus managed log persistence (30-day retention), a scripted acceptance checklist with expected outputs
- **Reliability:** 90-minute execution timeout kills wedged jobs; auto-start guarantees scheduled runs succeed even when the application has stopped
- **Security:** execution role trusted only by emr-serverless.amazonaws.com, scoped to exact S3 prefixes—no managed full-access policies

## Cost Notes

- us-east-1 rates: **$0.052624/vCPU-hour**, **$0.0057785/GB-hour** memory, first **20 GB** ephemeral disk per worker free
- Stated job (150 GB gzip JSON → Parquet, 13 workers, 22 min): **$1.44/run ≈ $43.32/month** nightly
- Minimal pre-init warm pool (12 vCPU / 48 GB): **$0.91/hour idle ≈ $663/month** if left running 24/7—only worth it above ~4 job starts/hour or hard sub-30-second start SLAs
- Full 13-worker fleet held warm 24/7: **~$3.94/hour ≈ $2,875/month**—the exact failure this kit prevents
- One-flag saving: `--architecture ARM64` (Graviton) cuts compute rates roughly 20% if your PySpark dependencies are pure Python
- Auto-stop tail after each warm-pool run: 15 idle minutes ≈ **$0.23**

## Troubleshooting

1. **Job stuck in SCHEDULED for minutes.** Cause: the account-level "Max concurrent vCPUs per account" Service Quota (often 16 on fresh accounts) or the application's `maximumCapacity` is below the job's 52-vCPU peak. → Fix: request a quota increase to 400 vCPUs in Service Quotas; confirm `maximumCapacity.cpu` ≥ peak demand.
2. **Executor OOM / container killed on a join.** Cause: 4 GB-per-core default ratio is too small for wide shuffles or skew. → Fix: move executors to 4 vCPU / 30 GB (`spark.executor.memory=27g`), raise `spark.sql.shuffle.partitions`, and let Spark 3.5 AQE skew-join handling work.
3. **"No space left on device" during shuffle.** Cause: 20 GB default ephemeral disk per worker exhausted by shuffle spill. → Fix: raise `spark.emr-serverless.executor.disk` (up to 200 GB per worker; storage beyond the free 20 GB is billed per GB-hour).
4. **Application never stops; idle charges appear.** Cause: pre-initialized capacity configured with auto-stop disabled or the idle timeout raised—warm workers bill with zero jobs running. → Fix: re-enable `autoStopConfiguration` with `idleTimeoutMinutes: 15` and verify via `get-application`.
5. **Job fails immediately with S3 AccessDeniedException.** Cause: execution role missing `s3:GetObject`/`s3:PutObject` on the exact prefixes, or trust policy missing emr-serverless.amazonaws.com. → Fix: apply the generated iam_policy.json verbatim; re-run `submit_job.sh`.
