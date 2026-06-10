# Nights and Weekends Off: EventBridge Scheduler + Lambda Auto-Stop for Non-Prod EC2, RDS, and ECS

Deploys a tag-driven EventBridge Scheduler + Lambda kit that powers non-prod EC2, RDS, and ECS down for the 108 idle hours in every 168-hour week—so your dev fleet bills like a part-time employee instead of a 24/7 one.

## The Problem

A week has 168 hours. Your team uses dev and staging for roughly 60 of them (07:00–19:00, Monday–Friday). The other 108 hours—64% of the week—those resources sit idle at full price.

Make it concrete. A typical 10-engineer startup's non-prod footprint in us-east-1:

- 12 × m5.large EC2 dev/staging instances at $0.096/hour → **$840.96/month** running 24/7
- 3 × db.m5.large RDS PostgreSQL (Single-AZ) at $0.192/hour → **≈$420.48/month**
- 2 ECS Fargate staging services, 8 tasks total at 1 vCPU / 2 GB (≈$0.0494/task-hour) → **≈$288/month**

That is ≈**$1,550/month of non-prod compute**, of which ≈**$1,000/month (~$12,000/year) is pure off-hours waste**. Most teams respond by deploying yet another waste *detector* that emails a report nobody reads. This prompt deploys the *fix*: a scheduler that actually stops the resources every night and weekend and starts them before standup—opt-in by tag, with a hard IAM-level guarantee that production is untouchable, and with the RDS 7-day auto-restart gotcha handled instead of silently re-billing you.

## Who This Is For

- Startups whose AWS bill is 30–50% non-prod and who have already been burned by "cost insight" tools that never act
- Platform/DevOps engineers who want a safe, reviewable, tag-based contract instead of a cron script someone wrote in 2022
- Engineering managers who need a one-week, zero-risk rollout path (dry-run first) before touching anything

## How to Use

1. Open Kiro CLI, Claude Code, or any AI coding assistant in an **empty directory**.
2. Paste the System Prompt below verbatim. Adjust the two cron lines and the timezone if your business hours differ from 07:00–19:00 America/New_York.
3. Let the assistant run its reality checks (account, region, current tagged inventory) before it writes any code—this is required by the prompt, not optional.
4. Review the generated Terraform—especially `iam.tf` (the explicit Deny prod guard) and the tag contract table in `README.md`.
5. Run `terraform init && terraform plan && terraform apply`. The Lambda deploys with `DRY_RUN=true`: for the first week it only logs what it *would* stop.
6. Tag one disposable instance with `autostop:schedule=office-hours`, watch the 19:00 dry-run log, then flip `dry_run = false` and roll the tag out to the rest of the fleet.

**Prerequisites**

- **Required Access:** An AWS account with permissions to create IAM roles/policies, Lambda functions, EventBridge Scheduler schedules, EventBridge rules, SNS topics, and CloudWatch log groups. AWS CLI v2 configured (`aws sts get-caller-identity` succeeds).
- **Recommended Background:** Basic Terraform (init/plan/apply), your org's tagging convention, and which non-prod resources are safe to stop.
- **Tools Required:** Kiro CLI or Claude Code; Terraform >= 1.5 with AWS provider >= 5.x; AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) so the assistant verifies EventBridge Scheduler cron syntax and RDS stop limitations against live docs instead of memory.

**Key Parameters:** Stop `cron(0 19 ? * MON-FRI *)` / start `cron(0 7 ? * MON-FRI *)` in America/New_York (DST-safe); opt-in tag `autostop:schedule=office-hours`; opt-out tag `autostop:exclude=true`; prod hard-stop on `Environment=prod|production` (code guard + IAM explicit Deny); `DRY_RUN=true` default; Lambda python3.13 / 256 MB / 120 s; CloudWatch log retention 30 days; SNS run-summary topic.

**Troubleshooting:** If the first dry run reports zero candidates, that's expected—the kit is strictly opt-in and touches nothing until you add the `autostop:schedule=office-hours` tag to at least one resource.

## System Prompt

```
# Non-Prod Off-Hours Scheduler — Infrastructure Build Request

## Project Overview
Build a production-ready, tag-driven off-hours scheduler that STOPS non-production EC2 instances, RDS/Aurora databases, and ECS services on nights and weekends and STARTS them before business hours. A 168-hour week contains 60 business hours (07:00-19:00, Mon-Fri); the other 108 hours (~64%) of always-on non-prod compute are waste. Target: cut tagged non-prod compute spend ~64% with zero human intervention after rollout. Deliver everything as Terraform (>= 1.5, AWS provider >= 5.x) plus one Python 3.13 Lambda. This is a scheduler that acts — not a detector that reports.

## Reality Checks — Do These FIRST, Before Generating Anything
1. Run `aws sts get-caller-identity` and `aws configure get region`. State the account ID and region you are building for. Never assume.
2. Inventory current candidates with the Resource Groups Tagging API (`tag:GetResources`, filter key `autostop:schedule`). Report how many EC2 instances, RDS instances/clusters, and ECS services are already tagged.
3. Verify EventBridge Scheduler cron syntax and the RDS stop limitations (Multi-AZ/replica constraints, the 7-day auto-restart) against current AWS documentation via aws_read_documentation before writing schedules.
4. If any check fails or the region lacks EventBridge Scheduler, STOP and report the blocker. Do not generate against assumptions.

## Detailed Requirements

### 1. Scheduling Core
- Two `aws_scheduler_schedule` resources: stop at `cron(0 19 ? * MON-FRI *)`, start at `cron(0 7 ? * MON-FRI *)`, with `schedule_expression_timezone = "America/New_York"` (DST-safe; do NOT use classic UTC-only EventBridge rules) and `flexible_time_window` mode OFF.
- No weekend start: Friday 19:00 stop holds until Monday 07:00 — 60 of 168 weekly hours running.
- Both schedules invoke one Lambda (python3.13, 256 MB, 120 s timeout) with payloads `{"action":"stop"}` / `{"action":"start"}`, via a Scheduler execution role allowing only `lambda:InvokeFunction` on that function ARN.

### 2. Tag Contract — Opt-In by Default
- ONLY resources tagged `autostop:schedule = office-hours` are ever touched.
- `autostop:exclude = true` overrides everything; skip with a logged reason.
- HARD STOP: any resource tagged `Environment` = `prod` or `production` is NEVER stopped or started, even if opted in. Enforce twice: (a) a code guard in the Lambda, and (b) an IAM explicit Deny on ec2:StopInstances, ec2:StartInstances, rds:StopDBInstance, rds:StartDBInstance, rds:StopDBCluster, rds:StartDBCluster with condition `"aws:ResourceTag/Environment": ["prod", "production"]`. The Deny must survive buggy code. This is non-negotiable.

### 3. Per-Service Behavior
- EC2: StopInstances/StartInstances in batches of 50 IDs with exponential backoff. Skip instance-store-backed instances (cannot stop) and anything carrying the `aws:autoscaling:groupName` tag — ASGs replace stopped instances; the README must recommend ASG scheduled actions for those instead.
- RDS: `stop_db_instance`/`start_db_instance` per instance; `stop_db_cluster`/`start_db_cluster` for Aurora. Skip instances with read replicas or in non-stoppable states (log the state). Handle the 7-day auto-restart: AWS restarts any RDS instance stopped 7 consecutive days. Create an EventBridge rule matching `{"source":["aws.rds"],"detail":{"EventID":["RDS-EVENT-0154"]}}` (started after exceeding max stop time) that re-invokes the Lambda in re-stop mode, AND make every nightly sweep idempotently re-stop anything AWS restarted — belt and suspenders.
- ECS: before scaling to 0, snapshot the live desiredCount into an `autostop:restore-count` tag on the service (ecs:TagResource); on start, restore exactly that count. Never overwrite the tag when desiredCount is already 0.

### 4. Safety and Operations
- `DRY_RUN` env var defaults to `"true"`: log every would-be action, change nothing. Enforcement is a one-variable flip.
- Idempotent: stop against stopped resources is a logged no-op; per-resource try/except so one failure never aborts the sweep.
- One structured JSON log line per resource: `{"resource":"...","action":"...","result":"...","reason":"..."}`; CloudWatch log group with 30-day retention.
- Publish one SNS run summary: stopped/started/skipped/failed counts plus estimated hourly savings; any failure count > 0 must be flagged in the subject line.
- Least-privilege IAM: enumerate every action (no wildcards on actions); add `"aws:ResourceTag/autostop:schedule": "office-hours"` conditions to all mutating EC2/RDS actions.

## Deliverables Requested
1. `main.tf`, `scheduler.tf`, `iam.tf`, `variables.tf`, `outputs.tf` (Terraform preferred)
2. `lambda/handler.py` — complete, no placeholders
3. `README.md` — tag contract table, one-week rollout plan (dry-run week 1), the 168h/50h savings math, and the monthly cost of the kit itself
4. `rollback.md` — how to start everything immediately and disable both schedules in under 2 minutes

## Acceptance Evidence — Emit With the Code
- Exact CLI commands to tag one test instance, invoke the Lambda with `{"action":"stop"}` via `aws lambda invoke`, and the expected JSON log line proving the dry run identified it
- A sample DRY_RUN summary log and a sample SNS message body
- The `terraform plan` resource count the user should expect

Align with the Cost Optimization, Operational Excellence, Security, and Sustainability pillars of the Well-Architected Framework. Output complete files without any preamble. The kit must be deployable in under 10 minutes.
```

## What You Get

1. **`main.tf`** — Lambda function (python3.13, 256 MB, 120 s), CloudWatch log group (30-day retention), SNS summary topic
2. **`scheduler.tf`** — two `aws_scheduler_schedule` resources (19:00 stop / 07:00 start, Mon–Fri, timezone-aware) plus the `aws.rds` EventBridge rule matching RDS-EVENT-0154 for 7-day auto-restart re-stops
3. **`iam.tf`** — least-privilege execution role with tag-conditioned Allows and the explicit Deny prod guard
4. **`variables.tf` / `outputs.tf`** — cron expressions, timezone, tag keys/values, `dry_run` flag, SNS subscription email
5. **`lambda/handler.py`** — complete handler covering EC2 batching, RDS instance + Aurora cluster stop/start, ECS desiredCount snapshot/restore, skip logic, structured logging
6. **`README.md`** — tag contract table, one-week dry-run rollout plan, savings math, kit running cost
7. **`rollback.md`** — the 2-minute "start everything now and disable schedules" procedure
8. **Acceptance evidence** — copy-paste verification commands and expected outputs

## Example Output

```text
{"resource": "i-0f3a9b2c1d4e5f607", "action": "stop", "result": "dry-run", "reason": "tag autostop:schedule=office-hours matched"}
{"resource": "arn:aws:rds:us-east-1:123456789012:db:staging-pg", "action": "stop", "result": "dry-run", "reason": "tag matched; state=available; no read replicas"}
{"resource": "arn:aws:ecs:us-east-1:123456789012:service/staging/api", "action": "stop", "result": "dry-run", "reason": "desiredCount=4 snapshotted to autostop:restore-count"}
{"resource": "i-0aa11bb22cc33dd44", "action": "skip", "result": "skipped", "reason": "member of ASG dev-asg — use scheduled scaling actions"}
SUMMARY action=stop candidates=17 stopped=0(dry-run:15) skipped=2 failed=0 est_hourly_savings_usd=1.99
```

## AWS Services Used

Amazon EventBridge Scheduler, AWS Lambda, Amazon EC2, Amazon RDS (including Aurora), Amazon ECS, AWS IAM, Amazon SNS, Amazon CloudWatch Logs, AWS Resource Groups Tagging API, Amazon EventBridge (RDS event rule)

## Well-Architected Alignment

- **Cost Optimization** — the entire point: reclaims the ~64% of weekly hours non-prod sits idle; emits estimated savings per run so the win is measurable
- **Operational Excellence** — dry-run-first rollout, structured JSON logs, SNS run summaries, a written rollback procedure, idempotent re-runs
- **Security** — least-privilege IAM with tag-scoped conditions on every mutating action; production protected by an explicit Deny that no code bug can bypass
- **Reliability** — per-resource error isolation, ASG/replica/instance-store skip rules, RDS 7-day auto-restart handled by event rule + nightly sweep
- **Sustainability** — 108 fewer compute-hours per resource per week is the most direct carbon reduction available in non-prod

## Cost Notes

- **Kit running cost:** ~44 schedule invocations/month at $1.00 per million EventBridge Scheduler invocations ≈ $0.00; Lambda ~44 × 30 s × 256 MB sits inside the free tier; CloudWatch Logs ingest well under $0.10/month; SNS email notifications free for the first 1,000/month. **Effectively $0.**
- **Savings (reference fleet, us-east-1):** 12 × m5.large ($0.096/h) + 3 × db.m5.large PostgreSQL ($0.192/h) + 8 Fargate tasks (≈$0.0494/h) ≈ $1,550/month at 24/7. At ~64% off-hours (108 of 168 weekly hours) → **≈$1,000/month back, ≈$12,000/year**.
- **What keeps billing:** stopped ≠ free. EBS volumes (gp3 $0.08/GB-month) and RDS storage + automated backups bill while stopped—12 × 100 GB gp3 is still $96/month. The README states this so finance isn't surprised.

## Troubleshooting

1. **A stopped RDS instance is running again mid-week.** Cause: AWS auto-restarts any RDS instance stopped for 7 consecutive days (event RDS-EVENT-0154)—common during holiday freezes. Fix: the kit's EventBridge rule re-stops it within minutes; confirm the rule is enabled and the nightly sweep logs show the re-stop.
2. **An EC2 instance stops, then a new one appears in its place.** Cause: the instance belongs to an Auto Scaling group, which replaces stopped members. Fix: the kit skips `aws:autoscaling:groupName`-tagged instances by design—use ASG scheduled actions (`MinSize=0` overnight) for those fleets.
3. **ECS service came back with desiredCount 0 or wrong size.** Cause: the `autostop:restore-count` tag was deleted or the service was manually scaled to 0 before the stop run. Fix: set the tag manually to the correct count and re-run start; the handler never overwrites the tag when desiredCount is already 0.
4. **AccessDenied stopping an instance that is tagged for autostop.** Cause: the resource also carries `Environment=prod`, so the explicit Deny fires. Working as intended—the prod guard is a hard stop. Fix: correct the resource's tags; never weaken the Deny.
5. **Schedules drift an hour after a DST change.** Cause: someone replaced the Scheduler schedules with classic UTC-only EventBridge cron rules. Fix: keep `aws_scheduler_schedule` with `schedule_expression_timezone`—EventBridge Scheduler shifts with the timezone automatically.
