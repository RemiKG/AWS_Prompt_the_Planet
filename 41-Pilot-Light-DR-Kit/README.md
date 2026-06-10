# Pilot-Light Cross-Region DR Kit: Route 53 Failover With S3/RDS/DynamoDB Replication and Game-Day-Measured RTO/RPO

Live cross-region data replication (RDS, DynamoDB, S3), Route 53 health-checked failover, and a scripted game day that prints your real RTO and RPO—regional-outage insurance for roughly 6% of what a duplicate stack costs.

## The Problem

The October 2025 us-east-1 event took thousands of products offline for most of a business day. The common responses—doubling the bill with an active/active second region, or writing a DR document nobody has ever executed—both fail. A DR plan without a measured RTO is a hypothesis, not a capability.

The math favors a middle path. For a typical $1,850/month single-region workload (ALB + ECS API, RDS PostgreSQL, DynamoDB, S3):

| Strategy | RTO | RPO | Standby cost/mo |
|---|---|---|---|
| Backup & restore | 4-8 hr | up to 24 hr | ~$35 |
| **Pilot light** | **25-30 min** | **seconds-minutes** | **~$108 (≈6%)** |
| Warm standby | < 10 min | seconds | ~$520 |
| Multi-site active/active | ~0 | ~0 | $1,900+ |

The replication lag you can actually buy today: DynamoDB Global Tables typically propagate writes in **under 1 second** (ReplicationLatency metric, milliseconds). RDS cross-region read replicas run **single-digit-second ReplicaLag** that spikes during write bursts and long transactions. S3 Cross-Region Replication with Replication Time Control replicates **99.99% of new objects within 15 minutes** (most within seconds), with an SLA backing the 99.9% threshold. Aurora Global Database holds typical **sub-second lag with a 1-second RPO design target**. Pilot light buys those RPO numbers without paying for idle compute.

What teams get wrong is not the replication—it's that they never run the failover, so the first test is a real disaster. This prompt builds the standby AND the timed game-day script whose pass/fail criterion is your measured RTO.

## Who This Is For

- Startups whose enterprise prospects send vendor questionnaires demanding documented, sub-hour DR.
- Teams running RDS + DynamoDB + S3 in one region who need provable RPO ≤ 5 minutes without warm-standby spend.
- Anyone who wrote "we have DR" in a compliance doc after October 2025 and has never measured a failover.

## How to Use

1. Copy the System Prompt below into Kiro CLI, Claude Code, or any AI assistant with AWS MCP tools (or save it as `.kiro/steering/pilot-light-dr.md`).
2. Replace every bracketed placeholder `[LIKE_THIS]`: regions, stack description, monthly spend, RTO/RPO targets, standby budget.
3. Run the first pass with **read-only credentials**—the assistant must inventory real resources before generating anything.
4. Review `dr-strategy-comparison.md` (if your RTO target is under 10 minutes, it will recommend warm standby instead—believe it).
5. Apply the Terraform replication layer, wait for RDS replica seeding and the S3 initial sync, then run `gameday.sh` in a low-traffic window.
6. File the emitted `MEASURED_RTO` / `MEASURED_RPO` as acceptance evidence. Re-run quarterly.

**Prerequisites**

- **Required Access**: AdministratorAccess (or PowerUserAccess plus IAM role creation) in the primary account; resource creation in the standby region; control of the Route 53 hosted zone. ReadOnlyAccess suffices for the inventory pass.
- **Recommended Background**: Terraform basics, DNS TTL behavior, RDS replication concepts.
- **Tools Required**: Terraform >= 1.7, AWS CLI v2, AWS Documentation MCP server (aws_read_documentation, aws_search_documentation), and use_aws (or AWS CLI) for live inventory.

**Key Parameters:** RTO target (30 min), RPO target (5 min), health-check interval (30s standard / 10s fast +$1/mo), failover record TTL (60s), ReplicaLag alarm (300s), DynamoDB ReplicationLatency alarm (15,000 ms), replica instance class (one size below primary), standby budget ($150 with 80%/100% alerts), game-day cadence (quarterly).

**Troubleshooting:** If the S3 replication rule fails to apply on first run, it's expected—Cross-Region Replication requires versioning enabled on BOTH source and destination buckets, and the prompt orders that step first; objects created before the rule need S3 Batch Replication to backfill.

## System Prompt

```
# Pilot-Light Disaster Recovery: Architecture & Failover-Test Request

## Project Overview
Design and implement a production-ready pilot-light disaster recovery setup for my existing single-region AWS workload, plus the scripted failover test that proves it works. Primary region: [PRIMARY_REGION, e.g. us-east-1]. Standby region: [STANDBY_REGION, e.g. us-west-2]. Workload: [DESCRIBE STACK, e.g. ALB + ECS Fargate API, RDS PostgreSQL 16 db.r6g.large with 200 GB gp3, one DynamoDB sessions table, one S3 asset bucket, Route 53 hosted zone for app.example.com]. Current spend: [$1,850]/month. Targets: RTO <= [30] minutes, RPO <= [5] minutes, standby running cost <= [10]% of primary monthly spend.

## Verify Reality First (hard requirements)
Before generating any code:
1. Inventory the actual resources with read-only calls: aws rds describe-db-instances, aws dynamodb describe-table, aws s3api get-bucket-versioning, aws route53 list-hosted-zones. Use real identifiers in every output file. Never invent ARNs, endpoints, or zone IDs.
2. Confirm the RDS engine and version support cross-region read replicas, and that the chosen replica instance class is offered in [STANDBY_REGION]. Check the AWS documentation, not memory.
3. Confirm S3 versioning is enabled on the source bucket (CRR requires it on both source and destination). If not, emit the versioning step FIRST and call out that pre-existing objects need S3 Batch Replication.
4. Report current DNS TTLs on the records that will become failover records.
If anything is missing or ambiguous, list the gaps and ask me before proceeding.

## Detailed Requirements

### 1. Strategy Justification
Produce a comparison table for THIS workload: backup & restore vs pilot light vs warm standby vs multi-site active/active, with realistic RTO, achievable RPO, and estimated monthly standby cost in dollars for each row. Recommend pilot light only if my RTO/RPO targets and budget actually support it; say so plainly if they don't.

### 2. Data Layer — continuous replication (the RPO layer)
- RDS: cross-region read replica in [STANDBY_REGION], one instance class below primary (scale up during failover); CloudWatch alarm on ReplicaLag > 300 seconds.
- DynamoDB: Global Tables replica in [STANDBY_REGION]; alarm on ReplicationLatency > 15,000 ms.
- S3: Cross-Region Replication with Replication Time Control (15-minute SLA backing 99.9% of objects, ~99.99% typical, $0.015/GB surcharge); alarms on OperationsPendingReplication and ReplicationLatency; least-privilege IAM replication role.
- AWS Secrets Manager multi-Region secret replication for every secret the app reads; ECR cross-region replication rule for container images.
- Emit a steady-state RPO table: expected replication lag per service, the CloudWatch metric that proves it, and the alarm threshold.

### 3. Failover Routing
Route 53 failover routing policy: PRIMARY alias record to the primary ALB with an HTTPS health check on /healthz (30-second interval, failure threshold 3), SECONDARY alias to the standby ALB with Evaluate Target Health enabled. Record TTL 60 seconds. Show the worst-case failover math: 3 x 30s detection + 60s resolver TTL ≈ traffic shifted in under 3 minutes.

### 4. Compute — off until needed (the cost layer)
Terraform module parameterized by region so the full app tier deploys into [STANDBY_REGION] with a single terraform apply -var="region=...". Pre-stage everything slow: replicated container images, copied launch templates or AMIs, VPC/subnets/security groups/IAM roles, and a minimal standby ALB so the SECONDARY record has a live target. Nothing billable runs in standby except the database replica, the idle ALB, and storage.

### 5. Failover Test — acceptance criteria
Write gameday.sh and failover-runbook.md. The script must: record T0; capture ReplicaLag, ReplicationLatency, and S3 pending-replication metrics at T0 (this is the measured RPO); simulate regional failure by failing the health check (never by deleting resources); promote the database with aws rds promote-read-replica and wait for available; scale the promoted instance up; apply the standby app stack; wait for the Route 53 flip; run smoke tests against the public hostname; then print MEASURED_RTO and MEASURED_RPO. Acceptance: MEASURED_RTO <= [30] minutes AND MEASURED_RPO <= [5] minutes, otherwise exit non-zero and name the slowest step. Include a per-step time budget in the runbook that sums under the RTO target, and a failback runbook (re-replicate back, planned cutover, restore failover records).

### 6. Cost Controls
Itemized monthly standby cost estimate per service. AWS Budgets monthly budget of [$150] filtered to [STANDBY_REGION] with alerts at 80% and 100%.

## Deliverables
1. ASCII architecture diagram covering both regions
2. dr-strategy-comparison.md
3. Terraform: replication.tf, route53-failover.tf, alarms.tf, budget.tf, and the standby-app/ module
4. failover-runbook.md with the timed step budget
5. gameday.sh emitting MEASURED_RTO / MEASURED_RPO with pass/fail exit code
6. failback-runbook.md
7. dr-cost-estimate.md

Follow the AWS Well-Architected Reliability pillar (REL13: define and test recovery objectives). The replication layer must apply in under 30 minutes; the full game day must complete and print its measurements in under one hour. Output the comparison table first, then the deliverables in order, without any preamble.
```

## What You Get

1. **dr-strategy-comparison.md** — the four strategies with RTO, RPO, and dollar costs computed for *your* workload, plus an explicit recommendation.
2. **replication.tf** — RDS cross-region read replica, DynamoDB Global Tables, S3 CRR with Replication Time Control and a least-privilege IAM role, Secrets Manager replicas, ECR replication rule.
3. **route53-failover.tf** — PRIMARY/SECONDARY failover records, /healthz health check (30s interval, threshold 3), 60-second TTLs.
4. **alarms.tf** — ReplicaLag > 300s, ReplicationLatency > 15,000 ms, OperationsPendingReplication, health-check status, all wired to SNS.
5. **budget.tf** — $150/month standby-region budget with 80%/100% alerts.
6. **standby-app/** — region-parameterized Terraform module that stands up the app tier with one apply.
7. **failover-runbook.md** + **failback-runbook.md** — timed, step-budgeted procedures.
8. **gameday.sh** — the acceptance test; prints MEASURED_RTO / MEASURED_RPO, exits non-zero on miss.

## Example Output

```
GAME DAY 2026-06-09 — target RTO 30m / RPO 5m
T0 14:02:10Z  health check failed (simulated)   ReplicaLag@T0: 11s
   14:13:42Z  rds promote-read-replica: available (+11m32s)
   14:21:05Z  standby stack healthy (terraform apply +7m23s)
   14:23:50Z  Route 53 flipped, public DNS resolves standby ALB
   14:25:51Z  smoke tests 8/8 passed
MEASURED_RTO=23m41s  PASS   MEASURED_RPO=11s  PASS
```

## AWS Services Used

Amazon Route 53 (failover routing + health checks), Amazon RDS (cross-region read replica, promotion), Amazon DynamoDB (Global Tables), Amazon S3 (CRR with Replication Time Control), AWS Secrets Manager (multi-Region secrets), Amazon ECR (image replication), Elastic Load Balancing (standby ALB target), Amazon CloudWatch (replication-lag alarms), AWS Budgets (standby cost guardrail), AWS IAM (least-privilege replication roles).

## Well-Architected Alignment

- **Reliability** — directly implements REL13 (Plan for Disaster Recovery): defined RTO/RPO objectives, a tested recovery path, replication alarms tied to the RPO target.
- **Operational Excellence** — runbooks are executable scripts with timestamps; quarterly game days produce acceptance evidence.
- **Cost Optimization** — the strategy table makes the cost/RTO trade explicit; compute stays off in standby; the replica runs one class down until promotion; Budgets alerts cap surprises.
- **Security** — secrets replicate through Secrets Manager, never baked into IaC; the S3 replication role is least-privilege; standby inherits the same security groups and IAM boundaries via the shared module.
- **Performance Efficiency** — right-sized standby with a documented scale-up step inside the RTO budget.

## Cost Notes

Standby-region steady-state for the reference workload ($1,850/mo primary, us-east-1 → us-west-2):

| Item | Monthly |
|---|---|
| RDS replica db.t4g.medium ($0.065/hr) | $47.45 |
| Replica storage 200 GB gp3 ($0.115/GB) | $23.00 |
| Standby ALB ($0.0225/hr, idle) | $16.43 |
| S3 standby storage 250 GB ($0.023/GB) | $5.75 |
| S3 RTC surcharge, 60 GB changed ($0.015/GB) | $0.90 |
| Inter-region transfer ~300 GB ($0.02/GB) | $6.00 |
| DynamoDB replicated writes (~1.5x write rate) | ~$5.00 |
| Route 53 health checks (2 × $0.50) | $1.00 |
| Secrets Manager replicas (6 × $0.40) | $2.40 |
| **Total** | **~$108 (≈6% of primary)** |

Game days add a few dollars per run (promoted instance hours + scale-up). RDS promotion is one-way—a full failback requires re-seeding a replica in the original direction.

## Troubleshooting

1. **S3 replication rule rejected on apply** — Cause: versioning disabled on source or destination bucket. → Fix: enable versioning on both first; backfill pre-existing objects with S3 Batch Replication—CRR is not retroactive.
2. **Route 53 never fails over during the drill** — Cause: the health check still passes (the ALB keeps answering after you stop instances), or resolvers cached an older, longer TTL. → Fix: simulate failure at the check target (block /healthz or invert the check); lower TTLs to 60s at least 24h before the first drill.
3. **RDS promotion blows the time budget** — Cause: promotion includes a replica reboot, and lag must drain first. → Fix: promote only when ReplicaLag is near zero in drills and pre-scale the replica; if still too slow, move to Aurora Global Database (sub-second lag, managed failover).
4. **terraform apply in standby fails on missing image/AMI** — Cause: ECR images and AMIs are regional. → Fix: enable ECR cross-region replication and copy AMIs in CI *before* relying on redeploy—pilot light dies without pre-staged artifacts.
5. **Standby bill creeps past $150** — Cause: RTC data charges and replica storage track primary growth. → Fix: the 80% Budgets alert exists for this; drop RTC (keep plain CRR) if a 15-minute S3 SLA exceeds your real RPO need.

---

**Bottom line:** replication layer applied in under 30 minutes, game day measured in under an hour, and a DR claim backed by a printed number instead of a diagram.
