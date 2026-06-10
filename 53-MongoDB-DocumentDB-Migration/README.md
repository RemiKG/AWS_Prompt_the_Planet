# MongoDB to Amazon DocumentDB Migration Runbook: Compatibility-Gated Cutover via AWS DMS or mongodump

Generates a complete, production-ready migration runbook — honest compatibility report, offline-vs-online decision with the math shown, index-first data load, and a validation query set that gates cutover — so your team moves to managed DocumentDB in one maintenance window instead of discovering broken queries in production.

## The Problem

Amazon DocumentDB is MongoDB **API-compatible** (it emulates the MongoDB 3.6, 4.0, 5.0, and 8.0 APIs on AWS's own distributed storage engine) — it is **not MongoDB**. Teams that treat it as a drop-in swap hit the same wall every time: the app connects, then writes fail with `{"ok":0,"errmsg":"Unrecognized field: 'txnNumber'"}` because modern drivers enable retryable writes by default and DocumentDB does not support them. Queries using `$elemMatch` inside `$all` return errors. Scripts relying on implicit sort order return shuffled results. The `admin` database doesn't exist, so user roles silently vanish from the dump.

The data load itself has a second trap: restoring 220 GB and *then* building 40 indexes can take 3x longer than necessary, because DocumentDB allows only **one index build per collection at a time**. AWS's documented best practice is the reverse of muscle memory — migrate index definitions to the empty target FIRST, then load data with `mongorestore --noIndexRestore`.

A migration without a compatibility gate and an acceptance test isn't a migration — it's a bet. This prompt makes the AI assistant run the gate first, choose the right migration path for your actual downtime budget, and refuse to declare success until a validation query set passes.

## Who This Is For

- Startups on self-managed MongoDB (EC2 or on-prem) or MongoDB Atlas consolidating onto AWS for VPC-native networking, IAM, KMS encryption, and one bill
- Teams at 100 GB–2 TB scale with a real downtime budget (minutes to a few hours) who need to choose between `mongodump/mongorestore` and AWS DMS with CDC — and want the decision justified with numbers
- Anyone who tried a naive lift-and-shift to DocumentDB once, got burned by an unsupported operator at 2 AM, and wants the honest unsupported-features list up front this time

## How to Use

1. Copy the System Prompt below into Kiro CLI, Claude Code, or any AI assistant configured with the AWS Documentation MCP server.
2. Replace every bracketed placeholder `[LIKE_THIS]` with your source MongoDB version, data size, collection count, downtime budget, and target region.
3. Run the prompt. The assistant produces `compatibility-report.md` FIRST — read it before provisioning anything. If it flags a hard blocker with no workaround, stop; that is the runbook working as designed.
4. Execute the generated artifacts in order: provision cluster → migrate indexes → load data (or start DMS tasks) → run `validate.js` → cut over using the checklist.

**Prerequisites**

- **Required Access:** Admin on the source MongoDB (read `mongod` logs, run `getIndexes()` and `rs.printReplicationInfo()`); AWS IAM permissions for Amazon DocumentDB control-plane actions (these use the `rds:` action namespace, e.g. `rds:CreateDBCluster`), AWS DMS, EC2 security groups and subnets, AWS Secrets Manager, and CloudWatch Logs.
- **Recommended Background:** Basic mongosh usage; familiarity with replica sets and the oplog; one prior VPC setup.
- **Tools Required:** AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`), mongosh, MongoDB Database Tools (AWS recommends versions up to and including 100.6.1 for DocumentDB), Python 3 with a clone of `awslabs/amazon-documentdb-tools` (compat-tool and index-tool), AWS CLI v2.

**Key Parameters:** Downtime budget (45 min), dataset size (220 GB), target engine API version (5.0), prod topology (3x db.r5.large across 3 AZs / dev: 1x db.t3.medium), backup retention (7 days), profiler slow-query threshold (100 ms), mongorestore insertion workers (8), validation sample (100 random docs per collection), source kept warm post-cutover (14 days).

**Troubleshooting:** If the first generated mongosh connection fails with TLS/SSL errors, it's expected — DocumentDB enforces TLS by default. Download the CA bundle from `https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem` and pass `--tlsCAFile global-bundle.pem`.

## System Prompt

```
# MongoDB to Amazon DocumentDB Migration Runbook Request

Act as a senior database migration engineer. Produce a complete, production-ready runbook for migrating our MongoDB workload to Amazon DocumentDB. Treat compatibility as a gate, not a footnote: Amazon DocumentDB emulates the MongoDB 3.6, 4.0, 5.0, and 8.0 APIs on its own storage engine — it is not MongoDB.

## Source Workload
- MongoDB [4.4] running as a [3-node replica set] on [EC2 / Atlas / on-prem]
- [220 GB] of data, [14] collections, largest collection [85 GB / 120M documents]
- Peak [4,000] ops/sec; downtime budget: [45 minutes]
- Drivers: [Node.js driver 6.x, PyMongo 4.x]; target region: [us-east-1]

## Verify Reality Before Generating (hard gate — do this first)
1. COMPATIBILITY CHECK FIRST. Direct me to run compat-tool (compat.py) from the awslabs/amazon-documentdb-tools repository against BOTH our application source code and our mongod logs. From its output plus my driver list, produce an honest unsupported-features table covering at minimum: retryable writes (DocumentDB does not support them — writes fail with "Unrecognized field: 'txnNumber'"; every connection string must set retryWrites=false), $elemMatch inside $all, correlated $lookup subqueries, reverse natural scans ({$natural: -1}), no admin or local databases (user roles are NOT dumped — they must be recreated on the target), no implicit result ordering (every order-dependent query needs an explicit sort()), and only one index build per collection at a time. For each finding: a tested workaround or HARD BLOCKER. If any hard blocker has no workaround, stop and say so plainly — do not generate a migration plan around it.
2. Use the AWS Documentation MCP server (aws_read_documentation) to confirm currently available DocumentDB engine versions and instance classes in my region before recommending any. Never invent flags, parameters, or limits.
3. Before proposing online migration, confirm the source oplog window (rs.printReplicationInfo()) exceeds the estimated full-load duration with 50% headroom.

## Detailed Requirements
### 1. Offline vs Online Decision
Apply this rule and show the math: choose OFFLINE (mongodump/mongorestore, MongoDB Database Tools <= 100.6.1) when the downtime budget >= estimated dump+restore time — plan around 30-60 GB/hour with 8 insertion workers over a 10 Gbps path, then verify with a timed 5 GB rehearsal. Otherwise choose ONLINE: AWS DMS full load + CDC in document mode (CDC requires the source to run as a replica set). Include the hybrid option (mongodump seed + DMS CDC catch-up) for datasets over 500 GB.

### 2. Target Cluster (production-ready)
[3] instances across 3 AZs, TLS enforced with global-bundle.pem, storage encrypted with AWS KMS, deletion protection enabled, automated backups with [7]-day retention, audit logs and the slow-query profiler ([100] ms threshold) exported to CloudWatch Logs, master credentials in AWS Secrets Manager, and a security group allowing port 27017 only from the app and migration-host security groups. DocumentDB is VPC-only — include the subnet group spanning 3 private subnets.

### 3. Index Rebuild Order
Migrate indexes BEFORE data: dump index definitions from the source with the Amazon DocumentDB Index Tool (index-tool from awslabs/amazon-documentdb-tools), restore them to the empty target cluster, then load data with mongorestore --noIndexRestore --gzip --numInsertionWorkersPerCollection=8. Flag any index types the Index Tool reports as unsupported.

### 4. Validation Query Set = Acceptance Criteria
Generate validate.js (mongosh) that must pass before cutover: (a) countDocuments() parity per collection, exact match; (b) aggregation checksums per collection ($sum over key numeric fields, min/max _id); (c) field-level diff of [100] random _ids per collection; (d) getIndexes() parity by name and key; (e) explain("executionStats") on our top [5] production query shapes with zero COLLSCAN stages; (f) a grep of app configuration proving retryWrites=false is set everywhere. Any failure aborts cutover.

### 5. Cutover and Rollback
Freeze writes, drain CDC lag to zero, run validate.js, flip the connection string stored in Secrets Manager, smoke-test, then keep the source running read-only for [14] days. Rollback = repoint the connection string to the source, executable in under 5 minutes, with named trigger conditions.

## Deliverables
1. compatibility-report.md — the honest unsupported-features list with workarounds
2. migration-decision.md — offline vs online, with the throughput math
3. Terraform for the DocumentDB cluster, subnet group, parameter group, and security group (Terraform preferred)
4. migrate-indexes.sh
5. migrate-data.sh (offline) or the DMS endpoint + task definitions (online)
6. validate.js
7. cutover-checklist.md with rollback triggers

Estimate the monthly cost of the target cluster and the one-time migration cost as part of the output. Align decisions with the Well-Architected Reliability and Security pillars and state how. Output the runbook directly without preamble. The migration must be executable by one engineer in a single maintenance window, and the validation suite must complete in under 15 minutes for 250 GB.
```

## What You Get

1. `compatibility-report.md` — every unsupported feature your workload actually touches, each marked *workaround* or *hard blocker*
2. `migration-decision.md` — offline vs online vs hybrid, justified against your downtime budget with throughput math and a rehearsal plan
3. Terraform for the DocumentDB cluster, subnet group, parameter group, security group, Secrets Manager secret, and CloudWatch log exports
4. `migrate-indexes.sh` — Index Tool dump/restore, run against the empty target
5. `migrate-data.sh` (mongodump/mongorestore with tuned flags) **or** DMS endpoint and task definitions with CDC
6. `validate.js` — the acceptance query set; cutover is blocked until it passes clean
7. `cutover-checklist.md` — write freeze, lag drain, connection-string flip, smoke tests, rollback triggers, 14-day source retention

## Example Output

> **COMPATIBILITY REPORT — gate: PASS with 3 workarounds, 0 hard blockers.** Scanned 212 source files and 48h of mongod logs with compat-tool (target API: 5.0). (1) Retryable writes enabled by default in PyMongo 4.x → append `retryWrites=false` to all connection strings (2 services, 3 strings). (2) `$elemMatch` within `$all` in `orders/service.py:141` → rewrite as `$and` of two `$elemMatch` clauses (equivalent, tested). (3) Nightly job sorts by `{$natural: -1}` → replace with `sort({_id: -1})`. **DECISION:** 220 GB at a measured 48 GB/hour ≈ 4.6 h restore > 45-minute budget → **ONLINE: DMS full load + CDC**. Oplog window 31.2 h ≥ 1.5 × full load. Proceeding to index migration: 38 of 38 index definitions supported.

## AWS Services Used

Amazon DocumentDB (with MongoDB compatibility), AWS Database Migration Service, Amazon VPC, Amazon EC2, AWS Secrets Manager, AWS Key Management Service, Amazon CloudWatch, AWS Identity and Access Management

## Well-Architected Alignment

- **Reliability:** Cutover gated on a deterministic validation suite; rollback path rehearsed and bounded at 5 minutes; source kept warm 14 days; oplog headroom verified before CDC starts.
- **Security:** TLS enforced with the RDS trust store bundle, KMS encryption at rest, VPC-only cluster with security groups scoped to port 27017 from named SGs, credentials in Secrets Manager, audit logs to CloudWatch.
- **Cost Optimization:** Right-sized dev (db.t3.medium) vs prod (db.r5.large x3) topologies; I/O-Optimized switch criterion stated; DMS instance sized to the window, not the maximum.
- **Operational Excellence:** Everything is a runbook artifact — repeatable, reviewable, and rehearsable against a staging cluster before production.
- **Performance Efficiency:** Index-first restore avoids serialized post-load builds; insertion workers tuned; explain() checks prove index usage before traffic arrives.

## Cost Notes

- **Dev/staging:** 1x db.t3.medium at $0.078/hour ≈ **$57/month** + storage at $0.10/GB-month.
- **Production:** 3x db.r5.large at $0.277/hour each ≈ **$607/month** compute; 220 GB storage ≈ $22/month; I/O at $0.20 per million requests. If I/O charges grow past roughly 25% of cluster spend, move to I/O-Optimized ($0.3047/hour per db.r5.large, $0.30/GB-month storage, zero per-request I/O charges).
- **Backups:** Free up to 100% of cluster storage within retention; beyond that, as low as $0.02/GB-month.
- **Migration one-time:** An m5.large jump host at $0.096/hour costs about $2.30 for a 24-hour offline run; a dms.t3.medium replication instance runs about $0.08/hour — and check the AWS DMS Free program, which has offered six months of free DMS use when the target is Amazon DocumentDB.
- **Hidden line item:** You run source and target in parallel for ~14 days. Budget one extra half-month of the old MongoDB bill and shut the source down on a calendared date.

## Troubleshooting

1. **Writes fail with `Unrecognized field: 'txnNumber'`** — Cause: drivers since MongoDB 4.2 enable retryable writes by default; DocumentDB doesn't support them. Fix: add `retryWrites=false` to every connection string (or the MongoClient constructor).
2. **mongorestore crawls for hours** — Cause: indexes restoring during data load, serialized by DocumentDB's one-build-per-collection limit. Fix: Index Tool first, then `--noIndexRestore`, and raise `--numInsertionWorkersPerCollection` to 8.
3. **DMS CDC task won't start or finds no changes** — Cause: source is standalone; CDC needs oplog access. Fix: convert the source to a single-node replica set (`rs.initiate()`), then restart the task.
4. **Connection refused / TLS handshake errors from mongosh** — Cause: TLS is enforced by default and the cluster is VPC-only. Fix: connect from inside the VPC (or via SSH tunnel) with `--tls --tlsCAFile global-bundle.pem`.
5. **Counts match but the app is slow post-cutover** — Cause: queries hitting COLLSCAN; sparse indexes need `$exists` in the query and `$regex` needs `hint()` to use an index on DocumentDB. Fix: rerun the explain() section of validate.js, add `$exists`/hints, and recheck index parity.
