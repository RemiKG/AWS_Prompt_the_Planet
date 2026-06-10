# Zero-Downtime RDS Major-Version Upgrades: Blue/Green Switchover Runbook for PostgreSQL and MySQL

Upgrade end-of-life PostgreSQL and MySQL on Amazon RDS through a guard-railed Blue/Green switchover with under a minute of write downtime—stop paying Extended Support and stop gambling production on in-place upgrades.

## The Problem

Major-version database upgrades are the change teams defer the longest, and the deferral now has a price tag. RDS for MySQL 5.7 left standard support in February 2024 and RDS for PostgreSQL 12 followed in February 2025; both bill **RDS Extended Support at $0.100 per vCPU-hour** (years 1–2), doubling to **$0.200 per vCPU-hour in year 3**. For a single 4-vCPU db.r6g.xlarge that is **$292/month—$3,504/year—rising to $584/month**, multiplied across every replica. The alternative most teams know, the in-place `modify-db-instance --engine-version` upgrade, takes the database fully offline—commonly 15–40 minutes—and is a one-way door with no tested rollback.

RDS Blue/Green Deployments is the built-in answer almost nobody uses: RDS clones your full topology (Multi-AZ, read replicas, storage) into a green staging environment on the new major version, keeps it in sync via replication, runs **switchover guardrails** (replication health, replica lag within limits, no active writes on green, no long-running writes or DDL on blue), and swaps DNS endpoints in typically under a minute—no application changes. If the switchover exceeds its timeout (default **300 seconds**, configurable **30–3,600 seconds**), RDS rolls everything back and neither environment changes. The feature itself costs $0; you pay only for the green resources while they exist.

Teams avoid it because three things are poorly understood: which engines actually support it, what the guardrails demand of you (DDL freeze, lag at zero, DNS TTL ≤ 5 seconds), and what rollback honestly looks like after switchover (replication stops—there is no reverse sync). This prompt generates a production-ready runbook that handles all three.

**The honest engine-support matrix** (verify per region before relying on it):

| Engine | Blue/Green support | Sync mechanism | Key prerequisite |
|---|---|---|---|
| RDS for MySQL | Yes (5.7, 8.0, 8.4 families) | Binary log replication | Automated backups on (enables binlog) |
| RDS for MariaDB | Yes (supported versions) | Binary log replication | Automated backups on |
| RDS for PostgreSQL | Yes (11.21+; major upgrades use logical replication) | Logical replication | `rds.logical_replication = 1` (static; reboot required) |
| Aurora MySQL | Yes (see Aurora guide) | Binary log replication | Binlog enabled |
| Aurora PostgreSQL | Yes (see Aurora guide) | Logical replication | Logical replication enabled |
| RDS for SQL Server / Oracle / Db2 | **No** | — | Use native tooling or DMS instead |

## Who This Is For

- Startups carrying EOL PostgreSQL 12 / MySQL 5.7 instances and bleeding Extended Support fees every month
- Platform and DBRE engineers who need a major-version upgrade with a written, rehearsed switchover plan instead of a maintenance-window prayer
- Teams who looked at DMS for an upgrade and want the simpler, managed, single-account path RDS already ships

## How to Use

1. Open your AI assistant with AWS credentials available—Kiro CLI, Claude Code, or Amazon Q Developer CLI all work.
2. Copy the System Prompt below into the session (or save it as `rds-blue-green-upgrade.md` in `.kiro/steering/` for Kiro).
3. Replace every bracketed placeholder `[LIKE_THIS]`: instance identifier, engine, current and target versions, region, traffic window.
4. Let the assistant run the reality checks first (`describe-db-instances`, `describe-db-engine-versions`)—do not let it generate a runbook for an instance it has not verified.
5. Review `preflight-report.md`; fix every FAIL (missing primary keys, logical replication off, DNS TTL) before creating the deployment.
6. Execute the runbook against a staging clone first, then production during your lowest-traffic window.

**Prerequisites**

- **Required Access:** IAM permissions for `rds:CreateBlueGreenDeployment`, `rds:SwitchoverBlueGreenDeployment`, `rds:DeleteBlueGreenDeployment`, `rds:DescribeBlueGreenDeployments`, `rds:DescribeDBInstances`, `rds:DescribeDBEngineVersions`, `rds:ModifyDBParameterGroup`, `rds:RebootDBInstance`, `cloudwatch:GetMetricData`, `events:PutRule`, and `sns:Publish` in the target account.
- **Recommended Background:** Working knowledge of MySQL binary log replication or PostgreSQL logical replication, RDS parameter groups, and your application's DNS caching behavior.
- **Tools Required:** AWS CLI v2, the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) for verifying current limitations, and `psql` or `mysql` client for lag queries.

**Key Parameters:** Switchover timeout (300s default, 30–3,600s range), replica-lag gate (ReplicaLag < 1s MySQL / `lsn_distance` = 0 PostgreSQL), client DNS TTL (≤ 5s), green soak window (72h), old-blue retention before deletion (7 days), target version (e.g., PostgreSQL 12.22 → 16.8, MySQL 5.7.44 → 8.0.x).

**Troubleshooting:** If a PostgreSQL deployment shows **Replication degraded**, it is expected—someone ran DDL or modified a large object on blue, which logical replication cannot carry. The only fix is "Delete with green databases" and recreate after enforcing a DDL freeze.

## System Prompt

```
# RDS Blue/Green Major-Version Upgrade — Runbook Generation Request

## Project Overview
Act as a senior database reliability engineer. Produce a production-ready, zero-downtime major-version upgrade plan for my Amazon RDS database using RDS Blue/Green Deployments, executed entirely with the AWS CLI. My database: [DB_INSTANCE_IDENTIFIER] running [ENGINE] [CURRENT_VERSION] in [REGION], targeting [TARGET_VERSION]. Traffic profile: [PEAK_WRITES_PER_SEC] peak writes/sec; lowest-traffic window [MAINTENANCE_WINDOW].

## Detailed Requirements

### 1. Verify Reality Before Generating (Error Management)
- Run `aws rds describe-db-instances --db-instance-identifier [DB_INSTANCE_IDENTIFIER]` and confirm the instance exists, plus its engine, version, Multi-AZ topology, read replicas, storage, and parameter groups. Never assume.
- Enforce the honest engine-support matrix: Blue/Green Deployments support RDS for MySQL, RDS for MariaDB, RDS for PostgreSQL, Aurora MySQL, and Aurora PostgreSQL only. RDS for SQL Server, Oracle, and Db2 are NOT supported — if the source is one of these, stop and say so instead of inventing a path.
- Validate the target with `aws rds describe-db-engine-versions --engine [ENGINE] --engine-version [CURRENT_VERSION] --query "DBEngineVersions[].ValidUpgradeTarget[].EngineVersion"`. If the target is not listed, propose the supported stepping-stone path (e.g., MySQL 5.7 -> 8.0 -> 8.4 as separate deployments).
- Check prerequisites: for RDS for PostgreSQL major upgrades, require `rds.logical_replication = 1` (static parameter — schedule the reboot) and flag every table lacking a primary key, because UPDATE/DELETE will not replicate without a PK or REPLICA IDENTITY FULL. For MySQL/MariaDB, confirm automated backups are enabled so binary logging is active.
- Verify Blue/Green availability in [REGION] and current limitations via the AWS Documentation MCP server (aws_read_documentation) before citing any behavior.

### 2. Green Environment Build
- Create the deployment: `aws rds create-blue-green-deployment --blue-green-deployment-name [NAME] --source [DB_ARN] --target-engine-version [TARGET_VERSION] --target-db-parameter-group-name [UPGRADED_PARAM_GROUP]`.
- Document that green is read-only by default, copies the full topology, and lazy-loads storage — include a hydration step (full-table reads) before any benchmark.
- For PostgreSQL: run ANALYZE on green after creation; logical replication does not carry optimizer statistics.

### 3. Pre-Switchover Gates (hard gates — all must pass)
- Replica lag at zero: CloudWatch `ReplicaLag` < 1 second on green (MySQL/MariaDB) or `lsn_distance` = 0 from `pg_replication_slots` plus `OldestReplicationSlotLag` near zero (PostgreSQL). Emit the exact SQL.
- DDL freeze on blue for the entire deployment window — on PostgreSQL, any DDL or large-object change degrades replication irreversibly and blocks switchover.
- Client DNS TTL ≤ 5 seconds, or Amazon RDS Proxy / a smart driver in front so connections re-route without waiting on DNS propagation (RDS runs extra proxy guardrail checks automatically).
- 48–72 hour soak running production-shaped read traffic against green.

### 4. Switchover Execution
- Execute `aws rds switchover-blue-green-deployment --blue-green-deployment-identifier [BGD_ID] --switchover-timeout 600` (range 30–3,600 seconds; default 300).
- Enumerate the built-in guardrails RDS runs: replication health, replica lag within limits, no active writes on green, no long-running writes or DDL on blue. State explicitly: if any guardrail fails or the timeout is exceeded, RDS rolls back and neither environment changes.
- Wire EventBridge blue/green deployment events to an SNS topic for SWITCHOVER_IN_PROGRESS and SWITCHOVER_COMPLETED, and poll `describe-blue-green-deployments` for status.

### 5. Rollback Reality — State It Honestly
- Before switchover: aborting is free. Delete the green environment; production is untouched.
- After switchover: replication between environments STOPS. The old blue (renamed -old1) is frozen at the switchover point; reverse replication does not exist. Rolling back means repointing applications and losing or manually reconciling every write made since switchover. The runbook MUST print this warning and define a 15-minute post-switchover go/no-go checklist (application error rate, p99 latency, ANALYZE completed, sequences verified) before declaring success or invoking rollback.

### 6. Cost Controls
- Quantify: green doubles instance and storage spend while it exists; the old blue keeps billing after switchover until deleted — schedule deletion after a 7-day regression window. Include the avoided Extended Support charge ($0.100/vCPU-hour, doubling in year 3) as the business case.

## Deliverables Requested
1. `preflight-report.md` — every reality check above with PASS/FAIL evidence and remediation
2. `upgrade-runbook.md` — numbered steps with exact CLI commands and expected outputs
3. `switchover.sh` — polls deployment status until AVAILABLE, checks lag gates, executes switchover with timeout
4. `verify-replication.sql` — lag queries for the engine in use
5. `rollback-decision-tree.md` — pre- and post-switchover branches with the data-loss warning in bold
6. `cost-estimate.md` — green-environment run cost vs. Extended Support avoided

Align every recommendation with the Well-Architected Reliability and Operational Excellence pillars. Output the artifacts directly without any preamble. Acceptance bar: an on-call engineer can execute the runbook end-to-end with under 60 seconds of write downtime, and every claim in it is backed by a command output in the preflight report.
```

## What You Get

1. **`preflight-report.md`** — engine/version eligibility, ValidUpgradeTarget proof, primary-key audit, logical-replication/binlog status, DNS TTL findings — each PASS/FAIL with the command that proved it
2. **`upgrade-runbook.md`** — the full numbered procedure: parameter-group prep, deployment creation, soak plan, gate checks, switchover, post-switchover verification
3. **`switchover.sh`** — guard-railed execution script that refuses to switch over while lag gates fail
4. **`verify-replication.sql`** — the `pg_replication_slots` lsn-distance query or MySQL replica-status checks, ready to paste
5. **`rollback-decision-tree.md`** — abort-before vs. reconcile-after branches with explicit data-loss consequences
6. **`cost-estimate.md`** — dollar comparison of the 72-hour green window vs. ongoing Extended Support

## Example Output

```
PREFLIGHT REPORT — orders-db (postgres 12.22 -> 16.8, us-east-1)
[PASS] Engine eligible: RDS for PostgreSQL supports Blue/Green (logical replication path)
[PASS] 16.8 listed in ValidUpgradeTarget for 12.22
[FAIL] rds.logical_replication = 0 — static parameter; set to 1 and reboot (est. 90s, schedule in window)
[FAIL] 2 tables without primary key: public.events_raw, public.audit_log
       -> UPDATE/DELETE will not replicate. Add PKs or SET REPLICA IDENTITY FULL before creating deployment.
[PASS] DNS TTL: app uses RDS Proxy (proxy guardrail checks will run at switchover)
[WARN] Flyway migrations auto-run on deploy — enforce DDL freeze for the deployment window
Gate to create-blue-green-deployment: BLOCKED until 2 FAILs are remediated.
```

## AWS Services Used

Amazon RDS (Blue/Green Deployments, parameter groups), Amazon RDS Proxy, Amazon CloudWatch, Amazon EventBridge, Amazon SNS, AWS CLI, AWS IAM, AWS Documentation MCP server

## Well-Architected Alignment

- **Reliability** — switchover guardrails plus hard lag gates; automatic rollback on timeout; topology (Multi-AZ, replicas) preserved in green; rehearsal on a staging clone required before production
- **Operational Excellence** — a written, evidence-backed runbook; EventBridge-to-SNS switchover notifications; a 15-minute go/no-go checklist instead of vibes
- **Cost Optimization** — eliminates $0.100–$0.200/vCPU-hour Extended Support; green window is time-boxed; old-blue deletion is scheduled, not forgotten
- **Performance Efficiency** — post-upgrade ANALYZE, storage hydration before benchmarking, 72-hour soak with production-shaped traffic
- **Security** — least-privilege IAM action list scoped to the upgrade; endpoints and TLS configuration unchanged through switchover

## Cost Notes

- **The feature is free.** You pay only for green-environment resources while they exist.
- Green clones your topology, so it roughly doubles instance + storage spend during the window: a Multi-AZ db.r6g.xlarge PostgreSQL green runs about $0.90/hour — a 72-hour soak costs roughly **$65**, plus ~$4.50 for 400 GB of gp3 ($0.115/GB-month) over the same window.
- **Extended Support avoided:** $0.100/vCPU-hour × 4 vCPUs × 730 hours = **$292/month ($3,504/year)** per db.r6g.xlarge, doubling to $584/month in year 3. The ~$70 upgrade window pays for itself in about a week.
- The old blue (`-old1`) keeps billing at standard rates after switchover (~$657/month for that Multi-AZ db.r6g.xlarge) — delete it after the 7-day regression window.
- Optional RDS Proxy: $0.015 per vCPU-hour of the target instance (about $44/month at 4 vCPUs) — worth it if you cannot control client DNS caching.

## Troubleshooting

1. **Deployment shows "Replication degraded" (PostgreSQL).** Cause: DDL or large-object changes ran on blue — logical replication cannot carry them, and switchover is blocked. Fix: choose "Delete with green databases," enforce a DDL freeze (pause migration tooling), and recreate the deployment.
2. **Switchover rolled back at the timeout.** Cause: replica lag exceeded the allowable limit for your timeout, or long-running writes/DDL held blue. Fix: terminate long-running transactions, retry in a quieter window, and raise `--switchover-timeout` toward 3,600s only after lag is at zero.
3. **Writes still hit the old blue after switchover.** Cause: client-side DNS caching above the 5-second RDS TTL (JVM defaults are a classic offender). Fix: set `networkaddress.cache.ttl=5` or front the database with RDS Proxy, which re-routes without DNS propagation.
4. **Green creation fails for PostgreSQL.** Cause: `rds.logical_replication` is 0 (static parameter) or tables lack primary keys, breaking UPDATE/DELETE replication. Fix: set the parameter to 1, reboot in a window, and add PKs or `REPLICA IDENTITY FULL` to the flagged tables.
5. **Queries are slow immediately after switchover (PostgreSQL).** Cause: logical replication does not transfer optimizer statistics, so the new primary starts with empty `pg_statistics`. Fix: run `ANALYZE` on the new production database as the first post-switchover runbook step.

Tested end to end: from pasted prompt to an executable, evidence-backed runbook in under 10 minutes.
