# Production ElastiCache for Valkey: Cluster-Mode Decision, a Real Failover Drill, and Stampede-Proof Caching

Stop praying your Redis survives an AZ loss and your cache doesn't melt under a thundering herd—ship a Multi-AZ ElastiCache cluster whose failover you have actually triggered and whose cache layer survives a stampede.

## The Problem

Caching is where startups quietly accumulate two kinds of debt, and both surface at the worst possible moment.

First, the high-availability story is fiction. A team enables Multi-AZ, sees a green console, and assumes they are covered. Nobody has ever forced a failover, so when an Availability Zone actually degrades at 2 AM, they discover their client library was caching the primary's IP, their connection pool never re-resolves the cluster endpoint, and a 15-30 second outage turns into a 20-minute one while engineers restart application pods by hand. Multi-AZ that has never been tested is not high availability—it is a hope.

Second, the cache itself becomes a denial-of-service weapon against your own database. A popular key with a 60-second TTL expires; in the same millisecond 4,000 concurrent requests all miss, all stampede the database to recompute the identical value, and the database that comfortably served 200 queries per second falls over at 4,000. This is the thundering-herd / cache-stampede problem, and a naive `GET` then `SET-on-miss` makes it inevitable.

This prompt closes both holes. It forces an explicit cluster-mode-enabled-versus-disabled decision tied to your dataset size and shard count, generates a Multi-AZ ElastiCache for Valkey (or Redis OSS) deployment, and—the acceptance test—runs `aws elasticache test-failover` against a real shard and measures recovery. It also emits production cache-strategy code with per-key TTL plus jitter, a single-flight lock so only one caller recomputes a missed key, and stale-while-revalidate so users never wait on a cold miss.

## Who This Is For

Founders and backend engineers who have outgrown a single Redis node and need a cache that survives an AZ loss without a 3 AM page; teams putting ElastiCache in front of RDS, Aurora, or DynamoDB to absorb read load; and anyone whose disaster-recovery story for their cache tier is currently "Multi-AZ is enabled, so we're fine." Intermediate level: you should know what a cache hit ratio is and have deployed at least one AWS resource.

## How to Use

1. Open Kiro CLI, Claude Code, or any AI assistant with AWS MCP access in the directory where you keep infrastructure code.
2. Paste the System Prompt below into a new session (or save it as `.kiro/steering/elasticache-valkey.md` with `inclusion: manual` so you can invoke it on demand).
3. Replace every bracketed placeholder—`[REGION]`, `[ENGINE]`, `[ENGINE_VERSION]`, `[DATASET_GB]`, `[PEAK_RPS]`, `[VPC_ID]`, `[APP_RUNTIME]`, `[MONTHLY_BUDGET]`—with your real values before sending.
4. Let the assistant verify reality first (engine version availability in the region, node-type availability, existing VPC and subnets across at least two AZs), then generate the cluster-mode decision table, Terraform, the failover drill, and the caching code.
5. Review the decision table and cost estimate, apply the Terraform, then run the generated `verify-failover.sh` to PROVE failover recovers within your target window.

Prerequisites:
- Required Access: an IAM principal with `elasticache:*`, `ec2:Describe*`, `ec2:AuthorizeSecurityGroupIngress`, `kms:CreateKey`/`kms:Decrypt`, `secretsmanager:CreateSecret`/`secretsmanager:GetSecretValue`, `cloudwatch:PutDashboard`, `cloudwatch:PutMetricAlarm`, and `iam:CreateServiceLinkedRole`. The AWS managed policy `AmazonElastiCacheFullAccess` plus CloudWatch and Secrets Manager access covers it.
- Recommended Background: Redis/Valkey basics (TTL, keyspace, `GET`/`SET`), VPC subnet and security-group fundamentals, Terraform basics.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) to confirm engine versions and node-type availability per region; AWS CLI v2 (`aws elasticache describe-cache-engine-versions`, `aws elasticache test-failover`); Terraform >= 1.6; a `valkey-cli` or `redis-cli` client.

Key Parameters: Engine Valkey 8.x (or Redis OSS 7.1) with `transit_encryption_enabled=true` and `at_rest_encryption_enabled=true`, `automatic_failover_enabled=true`, `multi_az_enabled=true`, `num_node_groups` (shards) and `replicas_per_node_group` (default 1 replica per shard for prod), `snapshot_retention_limit` 7 days, `snapshot_window` 03:00-05:00 UTC, `maintenance_window` Sun 05:00-06:00, default TTL 300s with +/-20% jitter, single-flight lock TTL 10s, stale-while-revalidate grace 60s, failover recovery target 30s, CloudWatch alarms on `DatabaseMemoryUsagePercentage` > 80%, `CPUUtilization` > 75%, `Evictions` > 0, and `CacheHitRate` < 0.8.

Troubleshooting: If the assistant generates a cluster-mode-enabled (sharded) configuration but your client uses a plain single-endpoint Redis driver, your reads will fail with `MOVED`/`CROSSSLOT` errors—this is expected, not a bug. Cluster mode enabled requires a cluster-aware client (e.g. `redis-py` `RedisCluster`, Lettuce cluster topology, ioredis cluster mode) and multi-key operations must use hash tags `{...}` to land keys on the same slot.

## System Prompt

```
# Production ElastiCache for Valkey / Redis OSS Deployment, Failover-Drill & Cache-Strategy Request

You are a senior AWS caching and reliability engineer. Produce a production-ready Amazon ElastiCache deployment that makes an explicit cluster-mode decision, is Multi-AZ with automatic failover, is encrypted in transit and at rest, is monitored, and—critically—whose failover you will PROVE by triggering `test-failover` against a real shard and measuring recovery. You will also emit production cache-strategy code that survives a cache stampede. Output Terraform (>= 1.6), shell scripts, and application code. No preamble.

## My Workload
- Region: [REGION] (e.g. us-east-1)
- Engine: [ENGINE] (Valkey or Redis OSS)
- Desired engine version: [ENGINE_VERSION] (e.g. Valkey 8.x, Redis OSS 7.1)
- Total cached dataset size: [DATASET_GB] GB
- Peak request rate: [PEAK_RPS] requests/second
- Existing network: [VPC_ID] with private subnets across >= 2 AZs
- Compute calling the cache: [APP_RUNTIME] (e.g. Lambda, Fargate, EC2, EKS)
- Monthly budget ceiling: $[MONTHLY_BUDGET]

## Verify Reality FIRST — do this before generating anything
1. Use aws_read_documentation / aws_search_documentation and `aws elasticache describe-cache-engine-versions --engine [ENGINE]` to CONFIRM [ENGINE_VERSION] exists in [REGION]. If it does not, stop and tell me the nearest supported version. NEVER invent a version.
2. Confirm the candidate node types (cache.t4g, cache.r7g, cache.m7g families) are offered in [REGION]. If unsure, query the API; do not guess a node type that may not exist.
3. Confirm [VPC_ID] exists and has >= 2 private subnets in distinct AZs via `aws ec2 describe-subnets`. Multi-AZ REQUIRES replicas in separate AZs—if only one AZ has subnets, STOP and tell me.
4. State every assumption you made. If you are not sure of a price, quota, or flag, query the API/docs and cite what you found rather than inventing it.

## Required Output 1 — Cluster-Mode Decision Table
Produce a markdown table that decides cluster mode enabled vs disabled for MY numbers, then recommend ONE and justify it in two sentences. Apply these rules:
- Cluster Mode DISABLED (single shard, one primary + read replicas): dataset fits in one node's memory (rule of thumb: dataset <= ~65% of node RAM to leave headroom for overhead and replication), write throughput fits one primary, and the client is a simple non-cluster driver. Simpler; vertical scaling only.
- Cluster Mode ENABLED (data sharded across `num_node_groups` shards, each with replicas): dataset exceeds one affordable node, OR write throughput exceeds one primary, OR you need to scale horizontally online. Requires a cluster-aware client and hash tags `{...}` for multi-key ops.
For each row give node type, memory, shards x replicas, total usable memory, and approximate on-demand $/month in [REGION]. Size so the working set is <= ~65% of usable memory. Recommend ONE configuration for [DATASET_GB] GB and [PEAK_RPS] RPS.

## Required Output 2 — Terraform
Generate Terraform for an aws_elasticache_replication_group:
- engine=[ENGINE], engine_version=the CONFIRMED version, node_type=the recommended class.
- automatic_failover_enabled=true and multi_az_enabled=true (HARD REQUIREMENT for prod).
- If cluster mode ENABLED: num_node_groups and replicas_per_node_group (>= 1 replica per shard). If DISABLED: num_cache_clusters >= 2 so there is at least one replica in another AZ.
- transit_encryption_enabled=true, at_rest_encryption_enabled=true with a customer-managed aws_kms_key, auth via a Valkey/Redis AUTH token or RBAC user stored in aws_secretsmanager_secret—NEVER hardcode the token.
- aws_elasticache_subnet_group across the private subnets; aws_security_group allowing 6379 ONLY from the [APP_RUNTIME] security group.
- snapshot_retention_limit=7, snapshot_window="03:00-05:00", maintenance_window="sun:05:00-sun:06:00", auto_minor_version_upgrade=true.
- aws_elasticache_parameter_group setting maxmemory-policy=allkeys-lru (or volatile-lru if you set TTLs on everything).
- aws_cloudwatch_dashboard with widgets: CacheHitRate, DatabaseMemoryUsagePercentage, CPUUtilization (use EngineCPUUtilization for cluster mode), Evictions, CurrConnections, ReplicationLag, NewConnections.
- aws_cloudwatch_metric_alarm for DatabaseMemoryUsagePercentage > 80% (3/5min), CPUUtilization > 75%, Evictions > 0, and CacheHitRate < 0.8.
- aws_budgets_budget at $[MONTHLY_BUDGET] with notifications at 80% and 100%.
Tag every resource with Environment, Owner, DataClassification.

## Required Output 3 — Failover Acceptance Test (this is the point)
Generate verify-failover.sh that PROVES Multi-AZ failover recovers, not that it is merely enabled:
1. Record the current primary node and its AZ via `aws elasticache describe-replication-groups`.
2. Start a background pinger from [APP_RUNTIME]'s perspective: in a tight loop, SET then GET a sentinel key through the configuration/primary endpoint and record every failure with a timestamp.
3. Trigger failover: `aws elasticache test-failover --replication-group-id <id> --node-group-id <shard-id>`. (Limit: up to 15 shards per rolling 24h; one replacement must finish before the next.)
4. Poll `aws elasticache describe-events` for "Failover from primary node ... to replica node ... completed" and `describe-replication-groups` until a NEW primary in a DIFFERENT AZ is current.
5. Stop the pinger. Compute downtime = (last failure timestamp - first failure timestamp); compute the failover wall-clock from test-failover call to new-primary-available.
6. Print a final ACCEPTANCE block: old primary + AZ, new primary + AZ, client-observed downtime in seconds, total failover duration. PASS if client-observed downtime <= 30s, FAIL otherwise. If FAIL, point at the likely cause (client not re-resolving the endpoint, no automatic_failover, single-AZ).
The drill costs nothing extra—test-failover uses your existing nodes.

## Required Output 4 — Stampede-Proof Cache-Strategy Code
Generate idiomatic [APP_RUNTIME]-language cache code (Python with redis-py, or Node with ioredis—match my runtime) implementing get_or_compute(key, compute_fn) with ALL of:
- TTL with JITTER: set TTL to base +/- up to 20% random so keys do not all expire in the same second.
- SINGLE-FLIGHT LOCK (stampede protection): on a miss, acquire a short lock (SET lock:<key> token NX EX 10). Only the lock winner runs compute_fn and writes the value; losers briefly wait and re-read, so the database sees ONE recompute, not [PEAK_RPS].
- STALE-WHILE-REVALIDATE: store value plus a soft-expiry. Serve stale data immediately when past soft-expiry while one background caller refreshes, so users never block on a cold miss.
- Release the lock with a check-and-delete (compare token) so you never delete someone else's lock; rely on lock TTL as a safety net.
For cluster mode enabled, use hash tags so lock and value share a slot: lock:{<key>}.
Include a comment stating the database-load math: without single-flight, a hot-key miss at [PEAK_RPS] RPS sends [PEAK_RPS] identical queries to the database; with it, exactly 1.

## Error Management
- If describe-cache-engine-versions has no matching version in [REGION], recommend the nearest and say so; do not silently substitute.
- If only one AZ has subnets, STOP—Multi-AZ is impossible; do not emit a single-AZ group claiming HA.
- If the replication group already exists, do NOT recreate—emit `terraform import` commands.
- If test-failover returns a throttling/limit error (15 shards / 24h), report it and retry guidance; do not mark the drill PASS.
- If the chosen node type is unavailable in [REGION], pick the nearest supported and state it.

## Cost Discipline
Estimate total monthly cost: (node $/hr x 730 x number of nodes) + backup storage beyond the free allotment + data transfer. Note that ElastiCache Serverless bills on data stored ($/GB-hour) plus ECPUs and can be cheaper for spiky/small workloads, while node-based is cheaper for steady large datasets. If the estimate exceeds $[MONTHLY_BUDGET], propose a smaller node type, fewer replicas (dev only), or Serverless, and re-state the table. Never provision more memory than ~1.5x the dataset justifies.

## Acceptance Bar
The deliverable is complete only when: terraform validate passes; the decision table names ONE recommended cluster-mode configuration with justification; verify-failover.sh has been run and prints client-observed downtime <= 30s; and the cache code demonstrably collapses a hot-key stampede to a single recompute. A developer should go from paste to a failover-proven, stampede-proof cache in under 30 minutes.
```

## What You Get

- `main.tf` — the ElastiCache for Valkey/Redis OSS replication group (Multi-AZ, automatic failover, cluster mode per the decision), KMS key, subnet group, security group, parameter group, and the AUTH token / RBAC user in Secrets Manager.
- `monitoring.tf` — CloudWatch dashboard, four named metric alarms (memory, CPU, evictions, hit rate), and the AWS Budgets budget with 80%/100% notifications.
- `variables.tf` / `terraform.tfvars.example` — every tunable surfaced (region, engine, version, dataset size, shards, replicas, TTL, budget).
- `cluster-mode-decision.md` — the enabled-vs-disabled table with node type, memory, shards x replicas, $/month, and the single recommended configuration.
- `verify-failover.sh` — the acceptance test that records the current primary, runs `aws elasticache test-failover`, measures client-observed downtime against the 30s bar, and prints the old/new primary and AZ.
- `cache.py` or `cache.js` — the stampede-proof `get_or_compute` with TTL jitter, single-flight lock, and stale-while-revalidate, hash-tagged for cluster mode.
- `connect.md` — how [APP_RUNTIME] connects via the configuration endpoint with TLS and the AUTH token, and which client library cluster mode requires.
- A monthly cost estimate and a stated-assumptions block.

## Example Output

Cluster-Mode Decision Table (excerpt):

| Config | Node type | Memory/node | Shards x replicas | Usable | ~$/mo (us-east-1, Valkey) |
|---|---|---|---|---|---|
| Cluster mode DISABLED | cache.r7g.large | ~13 GB | 1 x 2 replicas | ~13 GB | ~$385 |
| Cluster mode ENABLED | cache.r7g.large | ~13 GB | 3 x 1 replica | ~39 GB | ~$770 |

Recommendation: Cluster mode DISABLED, cache.r7g.large, 1 primary + 1 replica. Your 6 GB dataset fits comfortably under ~65% of a single node's ~13 GB, and 8,000 RPS is read-heavy and absorbed by the replica—sharding would add cluster-client complexity with no benefit.

ACCEPTANCE (from verify-failover.sh):
Old primary: my-cache-0001-001 (us-east-1a) | New primary: my-cache-0001-002 (us-east-1b)
test-failover -> new primary available: 21s | Client-observed downtime: 12s
[PASS] client-observed downtime 12s <= 30s target
Stampede test: 4,000 concurrent misses on hot key -> database saw 1 recompute (single-flight held).

## AWS Services Used

Amazon ElastiCache (Valkey / Redis OSS), Amazon ElastiCache Multi-AZ with automatic failover, AWS KMS, AWS Secrets Manager, Amazon CloudWatch (dashboards + alarms), AWS Budgets, Amazon VPC, AWS IAM. Typically fronting Amazon RDS / Amazon Aurora or Amazon DynamoDB.

## Well-Architected Alignment

- Reliability: Multi-AZ with `automatic_failover_enabled=true`, replicas in separate AZs, and an acceptance test that actually triggers `test-failover` and measures client-observed downtime against a 30s bar—turning assumed HA into measured HA.
- Security: TLS in transit (`transit_encryption_enabled=true`), encryption at rest with a customer-managed KMS key, AUTH token / RBAC stored in Secrets Manager (never hardcoded), and a security group that opens 6379 only to the app's security group.
- Performance Efficiency: the cluster-mode decision matches topology to dataset and throughput, `maxmemory-policy` is set explicitly, and TTL jitter plus stale-while-revalidate keep tail latency low.
- Cost Optimization: the right-sizing rule (working set <= ~65% of usable memory) avoids over-provisioning, a Budgets alarm fires at 80%/100%, and Serverless is offered for spiky workloads.
- Operational Excellence: a CloudWatch dashboard, four named alarms, and a repeatable scripted failover drill you can run on a schedule rather than discovering failure during a real incident.

## Cost Notes

All figures us-east-1, on-demand, June 2026; confirm against the ElastiCache pricing page before committing. Node-based pricing differs by engine: ElastiCache for Valkey runs roughly 33% cheaper than Redis OSS on the same node. cache.r7g.large (2 vCPU, ~13 GB) on-demand: about $0.175/hour for Valkey (about $128/month/node) versus about $0.26/hour for Redis OSS. A Multi-AZ disabled cluster mode group with 1 primary + 1 replica is two nodes, so roughly $256/month on Valkey. cache.t4g.medium (~3 GB) is a low-cost dev option at well under $50/month/node. ElastiCache Serverless for Valkey bills about $0.084/GB-hour for stored data plus about $0.0023 per million ECPUs and can be cheaper for small or spiky workloads—a 1 GB Serverless cache idling is on the order of $60/month with a 1 GB minimum. Backups: one automated backup is included; additional snapshot storage is about $0.085/GB-month. The failover drill costs nothing extra—`test-failover` reuses your existing nodes.

## Troubleshooting

- Client throws MOVED / CROSSSLOT errors after deploy. Cause: cluster mode is enabled but the application uses a plain single-endpoint Redis driver, or multi-key operations span slots. Fix: use a cluster-aware client (redis-py RedisCluster, Lettuce cluster topology, ioredis cluster mode) pointed at the configuration endpoint, and wrap related keys in a hash tag `{...}` so they share a slot.
- verify-failover.sh reports downtime far over 30s. Cause: the client caches the old primary's address and never re-resolves the endpoint, or there is no replica in a second AZ. Fix: connect through the configuration/primary endpoint (not a node IP), enable client-side topology refresh, and confirm `automatic_failover_enabled` and `multi_az_enabled` are both true with a replica in another AZ.
- test-failover returns a throttling or limit error. Cause: you exceeded 15 shards per rolling 24-hour window, or a prior node replacement has not finished. Fix: wait for the "Failover ... completed" event before the next call and stay within the 15-shard/24h limit; the script should not mark the drill PASS on this error.
- The database still gets hammered on a popular key despite caching. Cause: naive GET-then-SET-on-miss with no single-flight lock lets every concurrent miss recompute. Fix: use the generated get_or_compute with the SET lock NX EX single-flight lock plus stale-while-revalidate, so exactly one caller recomputes and the rest serve stale or wait briefly.
- TLS handshake or AUTH failures connecting from the app. Cause: `transit_encryption_enabled` is on but the client connects without TLS, or the AUTH token from Secrets Manager is wrong/rotated. Fix: enable TLS in the client, fetch the current token from Secrets Manager at startup (not a hardcoded copy), and confirm the security group allows 6379 from the app's security group.
