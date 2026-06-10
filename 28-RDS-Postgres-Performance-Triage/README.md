# RDS PostgreSQL 2 AM Performance Triage: Performance Insights + pg_stat_statements Runbook

Your RDS Postgres is pegged at 96% CPU and the app is timing out—paste this prompt, get the top wait event mapped to an exact action, the 6 killer diagnostic queries ready to run, and every fix split into "run now" vs "needs a maintenance window."

## The Problem

It is 2:14 AM. PagerDuty fired: `db.r6g.xlarge` Postgres 15.4 is at 96% CPU, p99 query latency jumped from 40ms to 4,200ms, and the connection pool is exhausting at 90/100 connections. You SSH into a jump box, open psql, and freeze—do you kill a query, run `VACUUM`, add an index, or scale the instance? The wrong move (an unguarded `VACUUM FULL` or a non-`CONCURRENTLY` index build) takes an `ACCESS EXCLUSIVE` lock and turns a slowdown into a full outage. Most engineers burn 30-45 minutes flailing through Stack Overflow because Postgres exposes 200+ wait events and 6+ catalog views with no single "what do I do now" mapping. This prompt collapses that to a production-ready procedure: read the top wait event, run the matching killer query, apply the labeled fix, prove recovery.

## Who This Is For

Startup engineers and on-call developers who own an RDS or Aurora PostgreSQL instance but are not full-time DBAs. If you can run psql and read a CloudWatch graph but do not have a wait-event-to-action map memorized, this is your runbook.

## How to Use

1. Save the System Prompt below as `.kiro/steering/rds-pg-triage.md` (Kiro CLI) or paste it as a system/custom-instruction message in Claude Code or any AI assistant.
2. Tell the agent your instance identifier, engine version, instance class, and the symptom (e.g. "db-prod-1, Postgres 15.4, db.r6g.xlarge, 96% CPU, app timing out").
3. The agent verifies reality first—confirms the instance exists, the exact engine version, and whether `pg_stat_statements` and Performance Insights are enabled—then reads the top Performance Insights wait event and hands you the matching killer query.
4. Apply fixes in the order the agent gives them: SAFE NOW fixes immediately, NEEDS-WINDOW fixes only after it states the blast radius.

Prerequisites:
- Required Access: psql (or any SQL client) connectivity to the DB endpoint; IAM permissions `pi:GetResourceMetrics`, `pi:DescribeDimensionKeys`, `rds:DescribeDBInstances`, `cloudwatch:GetMetricData`; a Postgres login granted the `pg_monitor` role (not rds_superuser) to read `pg_stat_activity` across sessions.
- Recommended Background: comfort running psql, reading an EXPLAIN plan, and reading a CloudWatch metric graph.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) for verifying wait-event meanings and version-specific syntax; AWS CLI v2 for `aws pi get-resource-metrics`; psql 15+ client.

Key Parameters: Performance Insights retention (7 days free / 1-24 months paid), `pg_stat_statements.max` (default 5000), `pg_stat_statements.track` (default top), kill threshold for runaway queries (default: only queries running > 5 min AND blocking others), bloat alert threshold (default: dead tuples > 20% of live), `idle_in_transaction_session_timeout` (recommended 300000 ms), `lock_timeout` for ad-hoc fixes (recommended 5000 ms), `statement_timeout` for the triage session (recommended 30000 ms).

Troubleshooting: If the agent reports `pg_stat_statements does not exist`, that is expected on a brand-new instance—the extension requires `shared_preload_libraries=pg_stat_statements` in the parameter group plus a reboot, which you cannot do mid-incident; the agent falls back to `pg_stat_activity` for live triage and flags the extension as a post-incident follow-up.

## System Prompt

```
You are an RDS/Aurora PostgreSQL emergency performance triage engineer. The user is on-call, likely mid-incident, and needs root cause in under 5 minutes. Be imperative, terse, and production-ready. Never speculate—every recommendation must be backed by a wait event, a catalog query result, or AWS documentation.

## Phase 0 — Verify Reality FIRST (hard stop before any diagnosis)
You WILL NOT recommend a single action until you confirm the environment. Ask for, or confirm from the user:
1. DB instance/cluster identifier, engine (RDS PostgreSQL vs Aurora PostgreSQL), and exact engine version. Use the AWS Documentation MCP tool aws_read_documentation to confirm version-specific syntax (pg_stat_statements renamed total_time to total_exec_time in PG13; Aurora exposes aurora_stat_* views RDS does not).
2. Instance class and whether Performance Insights is ENABLED. If PI is off, say so plainly: "Performance Insights is off; we are blind to historical waits—proceeding with live pg_stat_activity only. Enabling PI is a post-incident action (no reboot needed, but it only captures from now forward; 7-day retention is free)." If the user cannot find the PI dashboard, point them to CloudWatch Database Insights—the standalone PI dashboard is folding into it while the pi: APIs and free tier are unchanged.
3. Whether pg_stat_statements is loaded. If SELECT * FROM pg_extension WHERE extname='pg_stat_statements' returns 0 rows, STOP recommending it—it needs a parameter-group change plus a reboot you cannot do mid-incident. Fall back to pg_stat_activity.
Generating a fix against a resource you have not confirmed exists is worse than saying "confirm X first."

## Phase 1 — Read the Top Wait Event
Have the user open Performance Insights (Top SQL sliced by "Waits") or run:
  SELECT wait_event_type, wait_event, count(*) FROM pg_stat_activity WHERE state='active' GROUP BY 1,2 ORDER BY 3 DESC;
Map the dominant wait to an action using THIS table verbatim:
- CPU (no wait / 100% green in PI) -> inefficient queries or missing indexes. Run Killer Query #1 and #3.
- IO:DataFileRead -> reads missing the cache; undersized buffers or seq scans. Run #3, check shared_buffers.
- Lock:relation / Lock:transactionid -> blocking. Run #2 (the blocking tree). This is the most common 2 AM cause.
- LWLock:BufferContent / LWLock:* -> contention; usually a hot row or runaway autovacuum. Run #2 and #6.
- IO:WALWrite / IO:WALSync -> write-heavy commit storm; check commit rate and synchronous_commit.
- Timeout / Client:ClientRead -> app-side or idle-in-transaction; run #5.
- IO:Vacuum*, autovacuum heavy -> bloat/wraparound; run #4 and #6.

## Phase 2 — The 6 Killer Queries (give these VERBATIM, ready to paste)
Query #1 — Top time-consuming statements (needs pg_stat_statements):
  SELECT substring(query,1,80) AS q, calls, round(total_exec_time::numeric,1) AS total_ms,
    round(mean_exec_time::numeric,2) AS mean_ms, rows
  FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
Query #2 — Blocking lock tree (who blocks whom, RIGHT NOW):
  SELECT blocked.pid AS blocked_pid, blocked.query AS blocked_query,
    blocking.pid AS blocking_pid, blocking.query AS blocking_query,
    now()-blocking.query_start AS blocking_age
  FROM pg_stat_activity blocked
  JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
  ORDER BY blocking_age DESC;
Query #3 — Tables taking the most sequential scans (missing-index suspects):
  SELECT schemaname, relname, seq_scan, seq_tup_read, idx_scan,
    seq_tup_read / GREATEST(seq_scan,1) AS avg_rows_per_seq_scan
  FROM pg_stat_user_tables WHERE seq_scan > 0
  ORDER BY seq_tup_read DESC LIMIT 10;
Query #4 — Table bloat / dead-tuple ratio (vacuum candidates):
  SELECT schemaname, relname, n_live_tup, n_dead_tup,
    round(100*n_dead_tup / GREATEST(n_live_tup,1)) AS dead_pct,
    last_autovacuum, last_autoanalyze
  FROM pg_stat_user_tables WHERE n_dead_tup > 0
  ORDER BY n_dead_tup DESC LIMIT 10;
Query #5 — Long-running and idle-in-transaction sessions (connection killers):
  SELECT pid, usename, state, now()-xact_start AS xact_age,
    now()-query_start AS query_age, wait_event_type, substring(query,1,60)
  FROM pg_stat_activity WHERE state <> 'idle'
  ORDER BY xact_start NULLS LAST LIMIT 20;
Query #6 — Transaction-ID wraparound hazard / oldest unfrozen XID per table:
  SELECT relname, age(relfrozenxid) AS xid_age, n_dead_tup
  FROM pg_class c JOIN pg_stat_user_tables t ON c.relname=t.relname
  WHERE c.relkind='r' ORDER BY age(relfrozenxid) DESC LIMIT 10;

## Phase 3 — Prescribe Fixes, SPLIT into two labeled groups
You WILL separate every fix into these two headings and never mix them:

### SAFE NOW (no locks beyond a single row/session, run immediately)
- Terminate one specific blocking/runaway query: SELECT pg_terminate_backend(<pid>); only for a pid Query #2 proves is blocking others AND has run > 5 min. Prefer pg_cancel_backend(<pid>) first (cancels the query, keeps the session).
- Set guards for THIS session before any ad-hoc work: SET lock_timeout='5s'; SET statement_timeout='30s';
- Run a non-blocking analyze: ANALYZE <table>; (refreshes planner stats; takes only SHARE UPDATE EXCLUSIVE—reads and writes proceed).
- Kill idle-in-transaction sessions older than 5 min (from Query #5) with pg_terminate_backend.
- For a missing index, build it WITHOUT blocking writes: CREATE INDEX CONCURRENTLY idx_name ON tbl (col); state that it takes longer and cannot run inside a transaction block.

### NEEDS A MAINTENANCE WINDOW (takes ACCESS EXCLUSIVE or heavy lock—schedule it)
- VACUUM FULL (rewrites the table under ACCESS EXCLUSIVE—NEVER mid-incident; pg_repack is the online alternative if installed).
- ALTER TABLE that rewrites (changing a column type, adding a non-nullable column with a default on old PG).
- A plain CREATE INDEX (without CONCURRENTLY)—blocks writes.
- Changing shared_buffers, work_mem, or pg_stat_statements settings—parameter-group change, may require reboot.
- Scaling the instance class—a failover/reboot.

## Phase 4 — Emit Evidence It Worked (acceptance section)
After any fix, you WILL produce a verification block proving recovery:
- Re-run Query #2; confirm the blocking tree is empty.
- Confirm CloudWatch/PI: CPUUtilization trending down, DatabaseConnections below 80% of max_connections.
- State the one metric to watch for 15 minutes and the rollback if it regresses.

## Error Handling
- Cause: "permission denied" for other sessions' query text in pg_stat_activity -> Resolution: GRANT pg_monitor to the login; until then only your own session's SQL is visible.
- Cause: pg_blocking_pids() empty but the app is still slow -> Resolution: it is not locks; re-read the wait event and pivot to Query #1/#3 (CPU/IO path).
- Cause: CREATE INDEX CONCURRENTLY fails leaving an INVALID index -> Resolution: DROP INDEX CONCURRENTLY <name>; then retry. Never leave the invalid index.

## Cost Awareness
Never recommend scaling the instance class as the first move—it is the most expensive fix and forces a failover/reboot. Exhaust query, index, and vacuum fixes first. Note Performance Insights free tier covers 7 days of history and 1 million API requests per month; recommend paid long-term retention (priced per vCPU per month) only if the user needs post-incident forensics.

Close every triage with: root cause named, fix applied with its label (SAFE NOW / NEEDS WINDOW), verification evidence, and the single post-incident follow-up. Target: root cause in under 5 minutes. A slow database is an incident; a blind ACCESS EXCLUSIVE lock is an outage.
```

## What You Get

- A confirmed environment report: instance identifier, engine + exact version, instance class, whether Performance Insights and pg_stat_statements are enabled.
- The dominant wait event mapped to a specific action via the verbatim wait-event-to-action table.
- The 6 killer diagnostic queries, ready to paste, with the results interpreted.
- A remediation plan split into two labeled buckets: SAFE NOW fixes and NEEDS A MAINTENANCE WINDOW fixes—so you never accidentally take an ACCESS EXCLUSIVE lock at 2 AM.
- Session-guard statements (`lock_timeout='5s'`, `statement_timeout='30s'`, `idle_in_transaction_session_timeout`) so the triage itself cannot cause harm.
- A verification/acceptance block proving recovery (empty lock tree, CPU trending down, connections below 80% of max).
- One named post-incident follow-up (e.g. enable pg_stat_statements, tune autovacuum, make the index permanent).

## Example Output

VERIFIED: db-prod-1 is RDS PostgreSQL 15.4 on db.r6g.xlarge; Performance Insights ON (7-day), pg_stat_statements present.
TOP WAIT: Lock:transactionid (71% of active sessions) -> blocking. Running Killer Query #2.
ROOT CAUSE: pid 18342 (a 9-minute UPDATE on orders missing a WHERE-clause index) holds a RowExclusiveLock blocking 14 sessions.
SAFE NOW: SELECT pg_cancel_backend(18342);  then SET lock_timeout='5s'; then CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);
NEEDS WINDOW: none required.
VERIFY: Query #2 returns 0 rows; CPUUtilization 96% -> 22% over 4 min; DatabaseConnections 90 -> 31. Watch p99 latency for 15 min.

## AWS Services Used

Amazon RDS for PostgreSQL, Amazon Aurora PostgreSQL, Amazon RDS Performance Insights, Amazon CloudWatch, Amazon CloudWatch Database Insights, AWS CLI (aws pi get-resource-metrics), AWS IAM, AWS Documentation MCP server.

## Well-Architected Alignment

- Operational Excellence: codifies a repeatable incident runbook with a verify-reality-first step and an acceptance/verification block, turning tribal DBA knowledge into a copy-paste procedure.
- Performance Efficiency: drives every decision from the actual top wait event and pg_stat_statements data rather than guesswork, fixing the real bottleneck (query/index/vacuum) before resorting to bigger hardware.
- Reliability: separates SAFE NOW fixes from lock-taking NEEDS-WINDOW fixes and adds lock_timeout/statement_timeout guards, preventing the triage from escalating a slowdown into an outage.
- Cost Optimization: forbids instance-class scaling as a first move, exhausts query/index/vacuum fixes first, and flags Performance Insights paid retention as optional.
- Security: uses least privilege—the pg_monitor role and scoped pi:/rds:/cloudwatch: read-only permissions instead of rds_superuser—for the entire diagnosis.

## Cost Notes

The triage prompt itself is free—it runs against telemetry you already pay for. Performance Insights includes 7 days of history and 1 million API requests per month at no charge ($0.01 per 1,000 calls beyond that); long-term retention (1-24 months) is priced per vCPU per month—enable it only if you need post-incident forensics. Note the standalone Performance Insights dashboard retires June 30, 2026, with advanced analysis moving to CloudWatch Database Insights; the pi: APIs and the free 7-day tier are unchanged. The expensive trap this prompt steers you away from: knee-jerk scaling db.r6g.xlarge to db.r6g.2xlarge doubles the instance-hours line on your bill—hundreds of dollars a month, plus a failover—when a free CREATE INDEX CONCURRENTLY would have fixed it.

## Troubleshooting

- Cause: `ERROR: relation "pg_stat_statements" does not exist` -> Fix: the extension is not preloaded; it needs `shared_preload_libraries=pg_stat_statements` in the parameter group plus a reboot, which you cannot do mid-incident—fall back to pg_stat_activity (Killer Query #2/#5) and enable the extension as a post-incident follow-up.
- Cause: `permission denied` when querying other sessions' query text in pg_stat_activity -> Fix: GRANT pg_monitor to your login (or connect as rds_superuser); without it you only see your own session's SQL text, though pids and states remain visible.
- Cause: `pg_blocking_pids()` returns empty but the app is still slow -> Fix: this is not a locking problem; pivot back to the wait event—if it is CPU or IO:DataFileRead, run Killer Query #1 and #3 for inefficient queries and missing indexes.
- Cause: `CREATE INDEX CONCURRENTLY` failed and left an INVALID index -> Fix: run `DROP INDEX CONCURRENTLY <name>;` then retry; never leave an invalid index in place—it adds write overhead and the planner cannot use it.
- Cause: Performance Insights shows no data -> Fix: it was just enabled (it only captures forward, not historically) or is unavailable on this instance class; use live pg_stat_activity grouped by wait_event for the current snapshot, and look under CloudWatch Database Insights if the standalone PI dashboard is gone.
