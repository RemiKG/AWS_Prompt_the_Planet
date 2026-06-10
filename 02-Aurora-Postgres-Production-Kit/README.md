# Production Aurora PostgreSQL: Right-Sized Instances, RDS Proxy Pooling, and a Restore Actually Executed

Stop guessing your DB instance class and praying your backups work—ship a right-sized, connection-pooled Aurora PostgreSQL cluster whose restore has already been proven on a scratch instance.

## The Problem

Database failures are boringly predictable. A team picks `db.r6g.xlarge` "to be safe" and burns ~$400/month on a workload that fits a `db.r6g.large` at ~$200/month. Lambda fans out to 1,000 concurrent invocations, each opening its own connection, until the cluster throws `FATAL: remaining connection slots are reserved` and real users get 500s. And the classic: backups are "enabled," never restored, and the first test happens at 3 AM mid-incident—the snapshot restores but the application role, extensions, and search_path do not.

This prompt closes all three holes: a decision table tied to your actual workload, an RDS Proxy so Lambda shares a warm pool, and an acceptance test that restores your latest snapshot to a scratch cluster and asserts the data, roles, and extensions came back. No more "I think the backup works."

## Who This Is For

Founders and platform engineers who have outgrown a hand-clicked RDS instance, teams putting Lambda or Fargate in front of Postgres, and anyone whose disaster-recovery story is "we have snapshots, probably." Intermediate: basic SQL and Terraform.

## How to Use

1. Open Kiro CLI, Claude Code, or any AI assistant with AWS MCP access in your infrastructure-code directory.
2. Paste the System Prompt below into a new session, or save it as `.kiro/steering/aurora-postgres.md` with `inclusion: manual`.
3. Replace every bracketed placeholder—region, engine version, peak connections, working-set GB, budget, VPC, app runtime—with real values.
4. Let the assistant verify reality (engine version, classes, VPC/subnets) before generating anything.
5. Run `terraform validate` and `terraform apply`, then `verify-restore.sh`, and watch every assertion print PASS.

Prerequisites:
- Required Access: an IAM principal with `rds:*`, `secretsmanager:*`, `iam:CreateRole`/`iam:PassRole`, `ec2:Describe*`, and `backup:*`.
- Recommended Background: basic PostgreSQL (roles, extensions, `search_path`), Terraform, VPC subnet/security-group basics.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`); AWS CLI v2; Terraform >= 1.6; `psql`.

Key Parameters: Backup retention (7d dev / 35d prod, the Aurora maximum), RDS Proxy `idle_client_timeout` 1800s with `max_connections_percent` 90 / `max_idle_connections_percent` 50, Performance Insights 7d free tier, customer-managed KMS key, Serverless v2 0.5-8 ACU, scratch-restore `db.r6g.large` torn down on exit.

Troubleshooting: If Lambda cannot reach the generated RDS Proxy, the expected cause is subnet/security-group mismatch—the Proxy must share the cluster's VPC and allow the Lambda security group inbound on 5432. Not a prompt bug; the network path must be wired explicitly.

## System Prompt

```
# Production Aurora PostgreSQL Deployment & Restore-Verification Request

You are a senior AWS database engineer. Produce a production-ready Aurora PostgreSQL deployment right-sized to my workload, fronted by RDS Proxy, monitored with Performance Insights, backed up by automated snapshots plus an AWS Backup plan, and—critically—whose restore you PROVE by executing it against a scratch cluster. Output Terraform (>= 1.6) and shell scripts. No preamble.

## My Workload
- Region: [REGION]
- Engine version: [ENGINE_VERSION]
- Peak concurrent connections: [PEAK_CONCURRENT_CONNECTIONS]
- Hot working set: [WORKING_SET_GB] GB
- Compute in front: [APP_RUNTIME]
- Network: [VPC_ID], private subnets across >= 2 AZs
- Monthly budget ceiling: $[MONTHLY_BUDGET]

## Verify Reality FIRST
1. CONFIRM [ENGINE_VERSION] exists in [REGION] via aws_read_documentation / aws_search_documentation; if not, stop and name the nearest supported version.
2. CONFIRM candidate classes (db.r6g / db.r7g, Serverless v2) via aws rds describe-orderable-db-instance-options --engine aurora-postgresql.
3. CONFIRM [VPC_ID] has >= 2 private subnets in distinct AZs via aws ec2 describe-subnets; if not, stop and list what is missing.
4. Open with a Stated Assumptions block. NEVER invent an instance class, version, quota, or price—query the API or docs MCP and cite findings.

## Output 1 — Instance-Class Decision Table
Sizing rule: RAM >= 1.5x the hot working set. Minimum rows: <= 8 GB / < 100 conns / spiky -> Serverless v2 0.5-8 ACU; 8-16 GB / 100-300 conns -> db.r6g.large; 16-64 GB / 300-1,000 conns -> db.r6g.xlarge or db.r7g.xlarge; > 64 GB or heavy analytics -> db.r6g.2xlarge+. Per row: vCPU, RAM, ~$/month in [REGION], default max_connections (LEAST({DBInstanceClassMemory/9531392},5000)). Recommend exactly ONE row, justified in two sentences.

## Output 2 — Terraform
- aws_rds_cluster: storage_encrypted=true with a customer-managed aws_kms_key, backup_retention_period=35 prod / 7 dev, preferred_backup_window="03:00-04:00", deletion_protection=true, skip_final_snapshot=false + final_snapshot_identifier, enabled_cloudwatch_logs_exports=["postgresql"].
- aws_rds_cluster_instance: recommended class, performance_insights_enabled=true (retention 7), monitoring_interval=60 with monitoring_role_arn using AmazonRDSEnhancedMonitoringRole.
- aws_db_subnet_group; aws_security_group allowing 5432 ONLY from the app security group; aws_secretsmanager_secret for master credentials—NEVER hardcode a password.
- aws_db_proxy + default target group + target: engine_family=POSTGRESQL, require_tls=true, idle_client_timeout=1800, max_connections_percent=90, max_idle_connections_percent=50, auth via the secret through an IAM role scoped to secretsmanager:GetSecretValue on that ARN; comment why Lambda needs the Proxy (one connection per concurrent invocation).
- aws_backup_plan (daily, 35-day retention) + aws_backup_selection targeting the cluster ARN.
- aws_cloudwatch_dashboard (DatabaseConnections, CPUUtilization, FreeableMemory, Read/WriteLatency, ACUUtilization, DatabaseConnectionsBorrowLatency) and three alarms: CPU > 80%, FreeableMemory < 10% of RAM, DatabaseConnections > 80% of max_connections.
- aws_budgets_budget at $[MONTHLY_BUDGET], notifications at 80%/100%. Tag everything Environment, Owner, DataClassification.

## Output 3 — Restore Acceptance Test (the point)
verify-restore.sh must PROVE the backup works:
1. Find the latest automated snapshot (describe-db-cluster-snapshots --snapshot-type automated, newest SnapshotCreateTime).
2. restore-db-cluster-from-snapshot into <id>-restore-verify in the SAME subnet group, then create-db-instance to add one db.r6g.large—a snapshot restore creates the cluster, never its instances.
3. aws rds wait db-cluster-available, then db-instance-available.
4. psql assertions, each printing PASS/FAIL: known table count(*) > 0; application role exists in pg_roles; required extensions present in pg_extension.
5. Record snapshot age (measured RPO) and restore duration (measured RTO floor).
6. trap EXIT to ALWAYS delete the scratch instance and cluster, even on failure—the test costs only the restore window (~$0.28/hour).
End with an ACCEPTANCE block: snapshot id, RPO minutes, restore duration, every assertion's PASS/FAIL.

## Error Management
- No matching class in [REGION]: recommend the nearest family explicitly.
- Cluster already exists: emit terraform import commands—never recreate.
- RDS Proxy unsupported for the version/region: stop and report; never silently drop it.
- Proxy target unavailable: read describe-db-proxy-targets TargetHealth; fix the secret's credentials.
- Restore over 30 minutes: flag as an RPO/RTO risk in the acceptance block.

## Cost Discipline
Estimate monthly cost: instance + storage ($0.10/GB-mo) + I/O ($0.20/1M requests) + RDS Proxy ($0.015/vCPU-hr). If over $[MONTHLY_BUDGET], propose Serverless v2 or a smaller class and re-state the table. Never recommend a class larger than the working set justifies.

## Acceptance Bar
Complete only when terraform validate passes, the table names ONE recommended class with justification, and verify-restore.sh has run with every assertion printing PASS. Paste to restore-proven cluster in under 30 minutes.
```

## What You Get

- `main.tf` — cluster, instances, KMS key, subnet group, security group, secret, RDS Proxy + IAM role.
- `monitoring.tf` — CloudWatch dashboard, three named alarms, Budgets with 80%/100% notifications.
- `backup.tf` — AWS Backup plan (daily, 35-day retention) and selection.
- `variables.tf` / `terraform.tfvars.example` — every tunable surfaced.
- `instance-decision-table.md` — the workload-to-class table with the single recommended row.
- `verify-restore.sh` — the restore acceptance test with measured RPO/RTO and teardown on exit.
- `connect-via-proxy.md` — how apps connect through the Proxy endpoint.

## Example Output

| Workload | Class | vCPU | RAM | ~$/mo (us-east-1) | Default max_connections |
|---|---|---|---|---|---|
| 12 GB working set, ~250 conns, steady | db.r6g.large | 2 | 16 GB | ~$200 | ~1,700 |
| 40 GB working set, ~700 conns | db.r6g.xlarge | 4 | 32 GB | ~$400 | ~3,400 |

Recommendation: db.r6g.large. The 12 GB hot set fits inside 16 GB RAM (1.5x rule) and 250 peak connections sit well under the ~1,700 default, with RDS Proxy pooling Lambda bursts.

ACCEPTANCE (from verify-restore.sh):
Snapshot: rds:prod-aurora-2026-06-10-03-07 | RPO: 14 min | Restore duration: 9m41s
[PASS] orders table row count = 48,213 (> 0)
[PASS] role 'app' exists
[PASS] extension 'pg_stat_statements' present
Scratch cluster torn down. Test cost: ~$0.05.

## AWS Services Used

Amazon Aurora PostgreSQL, Amazon RDS Proxy, Amazon RDS Performance Insights, AWS Backup, AWS Secrets Manager, AWS KMS, Amazon CloudWatch, AWS Budgets, Amazon VPC, AWS IAM.

## Well-Architected Alignment

- Reliability: 35-day retention, an AWS Backup plan, Multi-AZ instances, a restore actually executed—assumed RPO/RTO becomes measured.
- Security: customer-managed KMS encryption, `require_tls=true`, credentials in Secrets Manager, 5432 open only to the app's security group.
- Cost Optimization: the decision table prevents over-provisioning; Budgets alerts at 80%/100%; Serverless v2 for spiky workloads.
- Performance Efficiency: Performance Insights, RAM >= 1.5x the working set, Proxy pooling that kills connection-storm latency.
- Operational Excellence: a dashboard, three named alarms, Enhanced Monitoring at 60s, a scripted restore drill you can schedule.

## Cost Notes

All figures us-east-1, on-demand, mid-2026, rounded. db.r6g.large: ~$0.28/hour, ~$200/month; db.r6g.xlarge: ~$0.55/hour, ~$400/month. Aurora Serverless v2: $0.12 per ACU-hour; a 0.5-ACU idle floor is ~$44/month. Storage: $0.10/GB-month plus $0.20 per 1M I/O requests—switch to I/O-Optimized when I/O dominates. RDS Proxy: $0.015 per vCPU-hour, ~$22/month for 2 vCPUs. Performance Insights: free at 7-day retention. The restore test costs a few cents at ~$0.28/hour.

## Troubleshooting

- Lambda cannot reach the database. Cause: function and Proxy in different subnets/security groups. Fix: place the Proxy in the cluster VPC's private subnets, allow the Lambda security group inbound on 5432, point the function at the Proxy endpoint.
- Restore script times out at the wait step. Cause: large snapshots exceed the default waiter (~20 min). Fix: loop `aws rds describe-db-clusters` until Status is available; surface the duration as an RTO data point.
- `terraform apply` says RDS Proxy is unavailable. Cause: Proxy support varies by engine version and region. Fix: check the RDS Proxy availability tables in the Amazon RDS User Guide via the docs MCP and pin a supported version.
- Proxy target never becomes available. Cause: secret credentials do not match a database user, or 5432 is blocked. Fix: read TargetHealth from `aws rds describe-db-proxy-targets`; correct the secret or security group.
