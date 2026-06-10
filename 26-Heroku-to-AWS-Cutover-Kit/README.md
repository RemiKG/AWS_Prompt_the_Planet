# Heroku to AWS Cutover Kit: Procfile/Config-Var Translation, DMS-Validated Database Move, and a Weighted Route 53 Cutover You Can Roll Back in 60 Seconds

Translate your Procfile and config vars into App Runner or ECS Fargate, replicate Heroku Postgres with a row-validated AWS DMS full-load-and-CDC task, and ramp Route 53 weighted DNS from 10% to 100%—so leaving Heroku is a boring afternoon, not a Sunday-night big-bang you cannot undo.

## The Problem

Heroku's free tier is gone, Eco dynos sleep, and the bill climbs fast: two Standard-2X dynos at $50 each plus a $50 Standard-0 Postgres is $150/month before the first worker, and real stacks with workers, Redis, and a bigger database clear $500. So teams decide to leave—and then freeze, because every Heroku exit looks like the same terrifying big-bang: pick a Sunday night, flip the DNS, and pray the database came across intact.

The failures are predictable and expensive. The Procfile `web: gunicorn app:app` and 40 config vars don't map cleanly to anything, so the app boots locally and 500s in the cloud because `DATABASE_URL` and `PORT` were never wired. The database "migration" is a `pg_dump`/`pg_restore` taken at 11 PM, but writes kept flowing on Heroku for the 90 minutes the restore took, so rows are silently lost and nobody validates counts. And the cutover is a single all-or-nothing DNS change behind a 3600-second TTL, so when something breaks, half the planet stays cached onto the dead origin for an hour and there is no rollback.

This prompt makes the exodus boring. It produces a Procfile and config-var translation table, an App Runner or ECS Fargate service that honors `PORT` and reads secrets from Secrets Manager, an AWS DMS full-load-plus-CDC migration that keeps Heroku and AWS in sync and validates every row, and a Route 53 weighted cutover that ramps 10 -> 25 -> 50 -> 100% with a documented 60-second rollback to weight 0.

## Who This Is For

Founders and engineers running a real app on Heroku (Rails, Django, Node, Flask, Go) with a Heroku Postgres database, who want off the PaaS without a multi-day outage or a lost-rows incident. Intermediate level: you should know your Procfile, have your config vars handy, and have deployed at least one AWS resource. If your database is tiny and read-mostly you can skip DMS; if it takes writes 24/7, the validated CDC path is exactly why this exists.

## How to Use

1. Open Kiro CLI, Claude Code, or any AI assistant with AWS MCP access in an empty `cutover-kit/` directory.
2. Paste the System Prompt below into a new session (or save it as `.kiro/steering/heroku-cutover.md` with `inclusion: manual`).
3. Gather your inputs from Heroku first: `heroku config -s -a <app>` (dumps every config var), your `Procfile`, `heroku pg:info -a <app>` (size, row estimate, Postgres version), `SHOW wal_level` via `heroku pg:psql` (CDC eligibility), and your custom domain.
4. Replace every bracketed placeholder—`[HEROKU_APP]`, `[RUNTIME]`, `[PROCFILE_CONTENTS]`, `[PG_VERSION]`, `[DB_SIZE_GB]`, `[REGION]`, `[APEX_DOMAIN]`, `[HOSTED_ZONE_ID]`, `[COMPUTE]` (apprunner | fargate), `[MONTHLY_BUDGET]`—with your real values.
5. Let the assistant verify reality first (RDS engine version in region, App Runner regional availability, CDC eligibility, hosted zone delegation), then generate the translation table, Terraform, the DMS task, and the weighted-cutover runbook.
6. Apply the Terraform, run the DMS task, wait for `ValidationState = Validated` on every table, then execute the cutover runbook one weight step at a time.

Prerequisites:
- Required Access: an IAM principal with `apprunner:*` or `ecs:*`+`elasticloadbalancing:*`, `rds:*`, `dms:*`, `iam:CreateRole`/`iam:PassRole`, `route53:*`, `secretsmanager:*`, `ec2:Describe*`, `logs:*`, `cloudwatch:PutDashboard`. The managed policies `AmazonRDSFullAccess`, `AmazonRoute53FullAccess`, and the DMS service role `dms-vpc-role` cover most of it.
- Recommended Background: you can read a Procfile, you have your Heroku config vars, and you own the domain in a Route 53 public hosted zone (or can move it there first).
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) to confirm RDS Postgres versions and DMS-supported endpoints per region; AWS CLI v2; Terraform >= 1.6; the Heroku CLI; `psql`.

Key Parameters: DMS migration type `full-load-and-cdc` (CDC requires `wal_level=logical` on the source), `EnableValidation=true` gate before the flip, replication instance `dms.t3.medium` (single-AZ for dev / Multi-AZ for prod), Route 53 weighted steps 10/25/50/100 with `set_identifier` "aws"/"heroku" and TTL 60s (NOT 3600), App Runner 1 vCPU / 2 GB default, Fargate 0.5 vCPU / 1 GB minimum, RDS `db.t4g.medium` Multi-AZ default with 7d (dev) / 35d (prod) backups, secrets in Secrets Manager (never in env files), rollback target weight 0 within 60s.

Troubleshooting: If a DMS table shows `ValidationState = No primary key`, that is expected and not a bug—DMS row validation requires a primary key or unique index, so tables without one (common for Rails join tables or event logs) will not validate. Add a primary key on the source, or accept those specific tables as full-load-only and validate them with manual `COUNT(*)` checks before the flip.

## System Prompt

```
# Heroku to AWS Cutover Kit — Migration Plan & Terraform Request

You are a senior AWS migration engineer. Produce a production-ready plan and Terraform (>= 1.6, AWS provider >= 5.0) that moves my app off Heroku onto AWS with ZERO lost rows and a reversible, weighted DNS cutover. Be opinionated, name exact resource types and flags, and refuse to emit anything you cannot verify. Output in fenced code blocks with no preamble.

## My App
- Heroku app: [HEROKU_APP]
- Runtime: [RUNTIME] (e.g. Rails 7 / Django 5 / Node 20 / Go 1.22)
- Procfile contents: [PROCFILE_CONTENTS]
- Heroku Postgres version: [PG_VERSION]   Database size: [DB_SIZE_GB] GB
- Target compute: [COMPUTE] (apprunner | fargate)
- Region: [REGION]   Apex domain: [APEX_DOMAIN]   Route 53 zone: [HOSTED_ZONE_ID]
- Monthly budget ceiling: $[MONTHLY_BUDGET]

## VERIFY REALITY FIRST — before generating anything
1. Use aws_read_documentation / aws_search_documentation to CONFIRM the RDS PostgreSQL engine version matching [PG_VERSION] is offered in [REGION]; if not, name the nearest supported version and STOP for my approval.
2. If [COMPUTE]=apprunner, confirm App Runner is offered in [REGION] — it is not available in every region. If absent, switch the plan to Fargate and say so.
3. CDC prerequisite: DMS change data capture from PostgreSQL requires wal_level=logical and a role with replication privileges. Have me check SHOW wal_level on the source via heroku pg:psql. If it is not logical and my plan cannot enable it, downgrade Deliverable 3 to full-load-only inside an explicit write-freeze window and SAY SO — never silently promise CDC.
4. Run `aws route53 get-hosted-zone --id [HOSTED_ZONE_ID]`; confirm the zone is PUBLIC and the registrar delegates to its nameservers. If not, STOP — the weighted cutover cannot work.
5. State every assumption. NEVER invent a service quota, instance class, version, or flag. If unsure, query the API or the docs MCP and cite the URL.

## Deliverable 1 — Procfile + Config-Var Translation Table
Produce a markdown table mapping EVERY Heroku concept to its AWS equivalent. Cover at minimum:
- Each Procfile process type. `web:` -> the [COMPUTE] start command; honor the dynamic `$PORT` (App Runner sets PORT, default 8080; Fargate via the container port). `worker:`/`clock:` -> a separate ECS service or EventBridge Scheduler task; NEVER assume one container runs both.
- `release:` (Heroku Release Phase) -> an ECS one-off task or CodeBuild step that runs schema migrations against RDS BEFORE any traffic shifts.
- `DATABASE_URL` -> the RDS endpoint, stored in Secrets Manager, injected at runtime — NOT baked into the image. `REDIS_URL` -> ElastiCache (or note none needed).
- Every other config var -> an App Runner runtime env var or an ECS `environment`/`secrets` entry; flag which values are SECRETS (API keys, signing keys) that MUST live in Secrets Manager, never plaintext.
For each row give: Heroku name, AWS target, where the value lives, and Secret? yes/no.

## Deliverable 2 — Compute (Terraform)
If [COMPUTE]=apprunner: aws_apprunner_service from an ECR image, 1 vCPU / 2 GB default, health check path /health, an auto-scaling configuration, runtime env vars plus secrets pulled through the instance role.
If [COMPUTE]=fargate: aws_ecs_cluster, aws_ecs_task_definition (0.5 vCPU / 1 GB minimum, awslogs driver, secrets via valueFrom Secrets Manager ARNs), aws_ecs_service behind an aws_lb (ALB) with target-group health checks, desired_count 2 across 2 AZs.
Either way: NEVER hardcode secrets — reference aws_secretsmanager_secret. Tag every resource Environment/Owner/DataClassification.

## Deliverable 3 — Database Migration with DMS + Validation (the part that prevents lost rows)
- aws_db_instance: PostgreSQL at the confirmed version, db.t4g.medium Multi-AZ default, storage_encrypted=true with a customer-managed aws_kms_key, backup_retention_period=35 (prod) / 7 (dev), deletion_protection=true.
- aws_dms_replication_instance (dms.t3.medium), source and target aws_dms_endpoint, and an aws_dms_replication_task with migration_type="full-load-and-cdc" so writes on Heroku keep replicating while we cut over.
- Enable validation in the task settings: {"ValidationSettings":{"EnableValidation":true}}. Explain that DMS compares rows source-to-target and reports ValidationState per table (Validated / Pending records / Mismatched records / No primary key), writing diagnostics to awsdms_control.awsdms_validation_failures_v1.
- Generate validate-cutover.sh: a gate that BLOCKS the flip until `aws dms describe-table-statistics` shows ValidationState=Validated and ValidationFailedRecords=0 for every table; surface any table stuck on "No primary key" as a manual COUNT(*) check. Note DMS suspends a table's validation after 10,000 failed records — treat that as a hard fail, not a warning.

## Deliverable 4 — Weighted Route 53 Cutover + Rollback (the reversible flip)
- Two aws_route53_record entries for [APEX_DOMAIN] sharing weighted_routing_policy, set_identifier "aws" and "heroku", TTL=60 (NOT 3600 — long TTLs turn a 60-second rollback into an hour).
- A runbook that ramps aws/heroku weight 10/90 -> 25/75 -> 50/50 -> 100/0, applying ONE step at a time and watching CloudWatch 5xx + p95 latency for a defined soak (default 15 min) between steps.
- An instant-rollback command printed prominently: set the aws weight to 0 (heroku 255) in a single apply; with TTL 60 the world drains off AWS in ~60 seconds.
- Only AFTER 100/0 holds clean for a full soak: stop the DMS task, run one final validation pass, then `heroku ps:scale web=0`.

## Error Management
- RDS version differs from Heroku's: recommend the nearest, and list extensions and `search_path` differences to re-check after migration.
- Table has no primary key: do NOT claim it validated — list it for manual verification.
- Hosted zone not authoritative for [APEX_DOMAIN]: STOP; weighted records will never take effect.
- CDCLatencySource above 60s: hold the cutover — the target is falling behind; recommend a larger replication instance before proceeding.
- Estimate exceeds $[MONTHLY_BUDGET]: propose App Runner over Fargate (no ALB/NAT) or a smaller RDS class and restate the number.

## Acceptance Evidence — prove it works
End with a "## Verification" section of literal commands and expected output: `aws dms describe-table-statistics` showing every table Validated with 0 failures; repeated `dig +short [APEX_DOMAIN]` returning answers from both origins in roughly the configured ratio; `curl -sf https://[APEX_DOMAIN]/health` returning 200 from the AWS origin; and the one-line rollback command. The measurable bar: a developer can execute this cutover one weight step at a time and roll back to Heroku in under 60 seconds at any step, with zero lost rows proven by DMS validation.

## Cost Discipline
Estimate monthly cost (compute + RDS + DMS instance + Route 53 + NAT if Fargate). DMS bills only while the replication instance runs — tear it down once 100/0 holds. Add aws_budgets_budget at $[MONTHLY_BUDGET] with 80%/100% alerts.
```

## What You Get

- `translation-table.md` — every Procfile process type and config var mapped to its App Runner / ECS Fargate equivalent, with a Secret? column flagging which values must live in Secrets Manager.
- `compute.tf` — either an `aws_apprunner_service` (ECR image, 1 vCPU / 2 GB, /health check, secrets via instance role) or an `aws_ecs_cluster` + `aws_ecs_task_definition` + `aws_ecs_service` behind an ALB across two AZs.
- `database.tf` — `aws_db_instance` (encrypted, Multi-AZ, 35-day backups) with a customer-managed `aws_kms_key` and the master secret in `aws_secretsmanager_secret`.
- `dms.tf` — `aws_dms_replication_instance`, source/target endpoints, and a `full-load-and-cdc` `aws_dms_replication_task` with `EnableValidation=true`.
- `dns.tf` — two weighted `aws_route53_record` entries (`set_identifier` "aws"/"heroku", TTL 60).
- `validate-cutover.sh` — gate script that blocks the flip until every table reports `ValidationState=Validated` with 0 failures.
- `cutover-runbook.md` — the 10 -> 25 -> 50 -> 100% ramp with soak windows and the one-line rollback-to-weight-0 command.
- `monitoring.tf` — CloudWatch dashboard (5xx, latency, DMS CDC latency, RDS connections) and an `aws_budgets_budget` with 80%/100% alerts.
- A monthly cost estimate and a stated-assumptions block.

## Example Output

Procfile + Config-Var Translation Table (excerpt):

| Heroku | AWS target | Where the value lives | Secret? |
|---|---|---|---|
| web: gunicorn app:app --bind 0.0.0.0:$PORT | App Runner start command, PORT=8080 | apprunner_service config | no |
| release: python manage.py migrate | ECS one-off task before traffic shift | run pre-cutover | no |
| DATABASE_URL | RDS Postgres endpoint | Secrets Manager arn:...:db-url | yes |
| SECRET_KEY_BASE | runtime secret | Secrets Manager arn:...:secret-key | yes |
| SENTRY_DSN | App Runner env var | apprunner runtime env | no |

DMS Validation (from validate-cutover.sh):
[PASS] table public.users        ValidationState=Validated  failed=0
[PASS] table public.orders       ValidationState=Validated  failed=0
[WARN] table public.events_join  ValidationState=No primary key -> manual COUNT(*): source=128401 target=128401 OK
Gate result: ALL VALIDATED. Cutover may proceed.

Cutover step 2 of 4: aws=25 / heroku=75 applied. Soaking 15m. 5xx rate 0.02%, p95 142ms. Rollback: terraform apply -var aws_weight=0 (drains in ~60s).

## AWS Services Used

AWS App Runner (or Amazon ECS on Fargate + Elastic Load Balancing), Amazon ECR, Amazon RDS for PostgreSQL, AWS Database Migration Service (DMS), Amazon Route 53 (weighted routing), AWS Secrets Manager, AWS KMS, Amazon CloudWatch, AWS Budgets, Amazon VPC, AWS IAM.

## Well-Architected Alignment

- Reliability: a `full-load-and-cdc` DMS task keeps source and target in sync so no writes are lost during cutover, per-table row validation proves the data arrived, and the weighted DNS ramp with 60s TTL makes every step reversible in about a minute instead of an hour.
- Security: secrets go to Secrets Manager and are injected at runtime (never baked into the image or plaintext env), RDS is encrypted with a customer-managed KMS key, and IAM is scoped to the listed actions.
- Cost Optimization: App Runner is offered as the cheaper path (no ALB or NAT Gateway), the DMS replication instance is torn down once 100/0 holds so it bills only during migration, and an AWS Budgets alarm fires at 80%/100%.
- Operational Excellence: a scripted cutover runbook with explicit soak windows, a CloudWatch dashboard, and a validation gate that blocks the flip turn a scary big-bang into a repeatable, observable procedure.
- Performance Efficiency: the prompt right-sizes compute (1 vCPU / 2 GB App Runner, 0.5 vCPU / 1 GB Fargate floor) to the workload and holds the cutover if DMS CDC latency climbs above 60s.

## Cost Notes

All figures us-east-1, June 2026. AWS App Runner: $0.064 per vCPU-hour (active) + $0.007 per GB-hour; a 1 vCPU / 2 GB service running continuously is roughly $0.078/hr ≈ $57/month, and provisioned-but-idle instances bill memory only at $0.007/GB-hour. ECS Fargate: $0.04048 per vCPU-hour + $0.004445 per GB-hour, so 0.5 vCPU / 1 GB is about $0.025/hr ≈ $18/month per task — plus an ALB at roughly $16-22/month and a NAT Gateway at about $32/month + $0.045/GB if tasks sit in private subnets. RDS db.t4g.medium Multi-AZ: about $0.13/hr ≈ $95/month including the standby, plus $0.115/GB-month gp3 storage. DMS dms.t3.medium: about $0.146/hr (Multi-AZ roughly double) — billed only while migrating, so a one-week cutover window costs $25-50 total, then the instance is deleted. Route 53: $0.50/month per hosted zone + $0.40 per million queries. Net effect: a Heroku stack costing $200-$500/month commonly lands on App Runner + RDS at $150-$200/month, with DMS a one-time line item.

## Troubleshooting

- App boots locally but 500s on App Runner. Cause: the service ignored Heroku's `$PORT` convention or never received `DATABASE_URL`. Fix: bind the web command to `0.0.0.0:8080` (App Runner's default PORT) and inject `DATABASE_URL` from Secrets Manager via the instance role, not a plaintext env var.
- The DMS full load completes but CDC never starts. Cause: change data capture from PostgreSQL requires `wal_level=logical` and a replication-capable role, and the source database does not allow it. Fix: check `SHOW wal_level` on Heroku; if your plan cannot enable it, run full-load-only inside an explicit write freeze (maintenance mode) and gate the flip on manual row counts.
- A DMS table never reaches Validated. Cause: it has no primary key or unique index, so DMS cannot row-validate it (`ValidationState=No primary key`). Fix: add a primary key on the source, or accept it as full-load-only and verify with manual `COUNT(*)` on source vs target before the flip.
- DMS reports Mismatched records or stops validating. Cause: ongoing writes during full load, a type or collation difference between source and target, or 10,000 failures hit the suspension limit. Fix: query `awsdms_control.awsdms_validation_failures_v1` for the failing keys, fix the root cause, and revalidate the affected tables.
- The cutover rolled back but users still hit the old origin. Cause: the weighted records were left at a long TTL (e.g. 3600s), so resolvers cache the old answer. Fix: set TTL=60 on both weighted records before starting the ramp so a rollback to weight 0 drains in about 60 seconds.
- CDC latency keeps climbing and the flip never feels safe. Cause: the source produces writes faster than `dms.t3.medium` can apply them (`CDCLatencySource` rising). Fix: hold the cutover, scale the replication instance up (or to Multi-AZ), and resume only once CDC latency settles under 60 seconds.
