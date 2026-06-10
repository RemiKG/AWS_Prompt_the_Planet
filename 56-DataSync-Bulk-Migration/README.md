# NAS-to-S3 Migration with AWS DataSync: Verified Cutover Without Saturating the Office Uplink

Plan and run a production-ready NAS-to-S3 migration with AWS DataSync—checksum-verified, bandwidth-throttled, resumable, and priced against Snowball before a single byte moves—so the office stays online and the cutover signs off with zero unverified files.

## The Problem

The lease on the colo is up, the NAS is out of warranty, and someone just typed `rsync -av /mnt/nas s3fs-mount/` and walked away. Three days later: the office internet has been unusable since Tuesday, the copy died at 61% with no record of which of the 12 million files made it, and nobody can say whether the bytes that did land in S3 are actually intact.

The real numbers are unforgiving. A 40 TB NAS over a 500 Mbps office link at 80% utilization takes roughly 9.3 days of *continuous* transfer—and that link is shared with payroll, video calls, and the CEO's demo. Do it wrong and you get all three classic failures at once: a saturated uplink during business hours, a non-resumable copy that restarts from zero after every blip, and a "migration complete" declaration backed by nothing but hope.

AWS DataSync solves all three—per-execution bandwidth throttling (`BytesPerSecond`), incremental `TransferMode=CHANGED` runs that resume by re-copying only what changed, and built-in end-to-end checksum verification (`VerifyMode`)—at a flat $0.0125/GB in Basic mode. But the console defaults won't schedule your nights-and-weekends window, won't tell you when a 210 TB Snowball Edge beats your thin pipe, and won't turn the verification report into a sign-off artifact. This prompt makes your AI assistant do exactly that.

## Who This Is For

- Startups at cohort zero with an office or colo NAS (NFS or SMB) that must land in S3 before a lease, warranty, or hardware dies
- IT leads at 20–200 person companies who own one shared internet link and cannot take it down during business hours
- Platform engineers who need a defensible acceptance gate ("zero verification errors on the final run") before decommissioning source storage
- Anyone deciding between DataSync over the wire and a Snowball Edge in a box—with math, not vibes

## How to Use

1. Copy the System Prompt below into Kiro CLI, Claude Code, Amazon Q Developer, or Cursor.
2. Replace every bracketed placeholder: `[TOTAL_TB]`, `[FILE_COUNT]`, `[NAS_HOSTNAME]`, `[NFS|SMB]`, `[BUCKET]`, `[REGION]`, `[LINK_MBPS]`, `[BUSINESS_HOURS]`, `[TZ]`.
3. Run it with AWS CLI credentials configured. The assistant verifies your bucket, region, and agent status first, then generates the six deliverables.
4. Deploy the DataSync agent VM on-prem (VMware ESXi, KVM, or Hyper-V) and activate it before running `provision-datasync.sh`.
5. Let nightly incrementals run for a week, then execute `cutover-runbook.md` over a weekend.

**Prerequisites**

- **Required Access:** AWS CLI v2 with permissions for `datasync:*`, `s3:CreateBucket`/`s3:PutObject` on the destination, `iam:PassRole` for the bucket-access role, and `logs:CreateLogGroup`. Root or admin access to the NAS export/share. Hypervisor access to deploy the agent VM (4 vCPU / 32 GB RAM / 80 GB disk; 64 GB RAM if the tree exceeds 20 million files).
- **Recommended Background:** Basic NFS/SMB administration; familiarity with S3 storage classes; one read of the DataSync task-execution lifecycle.
- **Tools Required:** AWS CLI v2, the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) so the assistant verifies current quotas and pricing instead of guessing, and outbound TCP 443 from the agent to AWS endpoints.

**Key Parameters:** Throttle (`BytesPerSecond=12500000` ≈ 100 Mbps business hours, `-1` unlimited overnight), VerifyMode (`ONLY_FILES_TRANSFERRED` nightly / `POINT_IN_TIME_CONSISTENT` final cutover), schedule (`cron(0 3 * * ? *)`, 1-hour minimum interval), excludes (`*/.snapshot|*.tmp|*~|/lost+found`), S3 storage class (STANDARD at ingest), log retention (30 days), Basic-mode quota (50M files/execution).

**Troubleshooting:** If the first execution fails during the PREPARING phase on a large tree, it's expected—the agent VM is under-provisioned for the file count. Trees over 20 million files need a 64 GB RAM agent, and Basic mode hard-caps at 50 million files per execution; split larger trees into per-share tasks with include filters.

## System Prompt

```
# NAS-to-S3 Migration Design Request: AWS DataSync, Verified and Resumable

## Project Overview

Act as a senior storage migration engineer. Design and script a production-ready
migration of [TOTAL_TB] TB / [FILE_COUNT] files from an on-premises NAS exporting
[NFS|SMB] shares at [NAS_HOSTNAME] to s3://[BUCKET] in [REGION], over a
[LINK_MBPS] Mbps internet link shared with a working office. The link must never
be saturated during business hours ([BUSINESS_HOURS] [TZ]), the transfer must
survive interruption without restarting from zero, and cutover sign-off must be
backed by DataSync's built-in verification—not trust.

## Detailed Requirements

### 1. Verify Reality First — before generating anything
- Confirm the destination exists and DataSync is available in [REGION]:
  run `aws s3api head-bucket --bucket [BUCKET]` and `aws datasync list-agents`.
  If no agent is ONLINE, stop and output agent VM deployment + activation steps
  instead of the migration scripts.
- Use the AWS Documentation MCP server (aws_read_documentation) to confirm
  current quotas: Basic mode supports 50 million files per task execution and
  10 Gbps max per agent task; if [FILE_COUNT] > 20 million, require a 64 GB RAM
  agent (otherwise 4 vCPU / 32 GB RAM / 80 GB disk).
- Confirm outbound TCP 443 from the agent subnet; offer the VPC-endpoint
  activation path if traffic must stay private.

### 2. Transfer Architecture
- One Basic mode task per top-level share with TransferMode=CHANGED so every
  nightly run is incremental and any failure resumes by re-copying only deltas.
- Source location via create-location-nfs (or create-location-smb with the
  password stored in AWS Secrets Manager, never inline).
- Destination via create-location-s3 with --s3-storage-class STANDARD and a
  least-privilege BucketAccessRoleArn scoped to s3://[BUCKET]/* only.
- CloudWatch log group /aws/datasync with LogLevel=TRANSFER, 30-day retention.

### 3. Throttle and Schedule
- Business hours: Options.BytesPerSecond=12500000 (100 Mbps). Nights and
  weekends: raise to -1 (unlimited) with update-task-execution on the RUNNING
  execution—BytesPerSecond is the only option changeable mid-run; no restart.
- ScheduleExpression="cron(0 3 * * ? *)" for nightly incrementals; note the
  1-hour minimum schedule interval and the 50-queued-executions cap per task.

### 4. Verification = Acceptance
- Nightly runs: VerifyMode=ONLY_FILES_TRANSFERRED (checksum every file moved).
- Final cutover run: VerifyMode=POINT_IN_TIME_CONSISTENT (full source vs
  destination scan). Acceptance criterion: zero verification errors.
- Enable --task-report-config writing SUCCESSES_AND_ERRORS reports to
  s3://[BUCKET]-reports/; that report IS the sign-off artifact.

### 5. Scope Filters
- Excludes: FilterType=SIMPLE_PATTERN,
  Value="*/.snapshot|*.tmp|*~|/lost+found|*.bak" (pipe-delimited).
- Includes per migration wave, e.g. Value="/finance|/engineering" for wave 1.

### 6. Cost and the Snowball Break-Even
- Price it: DataSync Basic $0.0125/GB transferred, plus S3 PUT requests
  ($0.005 per 1,000) and S3 Standard storage ($0.023/GB-month).
- Compute network days = TB × 116 ÷ usable Mbps (assume 80% utilization). If
  that exceeds 14 days, recommend AWS Snowball Edge Storage Optimized 210 TB
  instead: $1,800/job up to 100 TB or $3,200 for 101–210 TB (US East), 15
  onsite days included, $250/day after, $0.00/GB import into S3, shipping
  extra. Present both totals (dollars AND calendar days) side by side.

### 7. Hybrid Tail
- If some users still need an SMB/NFS path after cutover, specify an Amazon S3
  File Gateway ($0.01/GB written by the gateway, capped at $125/month per
  gateway) instead of keeping the NAS alive.

## Deliverables Requested

1. migration-plan.md — waves, timeline, throttle calendar, owner per step.
2. provision-datasync.sh — idempotent AWS CLI: locations, tasks, schedule,
   filters, logging, task reports.
3. throttle.sh — clamps to 100 Mbps at 08:00, releases to unlimited at 18:00
   via update-task-execution.
4. verify-acceptance.sh — parses the final task report from S3; exits non-zero
   on any verification error and prints the failed-file list.
5. cost-comparison.md — DataSync vs Snowball math with my numbers substituted.
6. cutover-runbook.md — read-only freeze on the NAS, final POINT_IN_TIME_CONSISTENT
   run, mount/DNS flip, rollback trigger, decommission checklist.

## Error Management
- Never set PreserveDeletedFiles=REMOVE before the final run is verified.
- If any execution reports verification errors, halt the schedule, emit the
  failed-file list, and do not declare success.
- If estimated network days > 14, stop and present the Snowball option with
  break-even math before scripting anything.

Output the deliverables directly without any preamble. The scripts must run
against a live account in under 15 minutes, and cutover must complete with
zero unverified bytes.
```

## What You Get

1. **migration-plan.md** — wave-by-wave plan with a throttle calendar (100 Mbps 08:00–18:00, unlimited nights/weekends) and per-step owners
2. **provision-datasync.sh** — idempotent AWS CLI script creating both locations, the task(s), `cron(0 3 * * ? *)` schedule, include/exclude filters, CloudWatch logging, and S3 task reports
3. **throttle.sh** — two-line cron-able clamp/release using `aws datasync update-task-execution --options BytesPerSecond=...`
4. **verify-acceptance.sh** — turns the final task report into a pass/fail gate; non-zero exit on any verification error
5. **cost-comparison.md** — DataSync-vs-Snowball dollars and calendar days for your exact TB and Mbps
6. **cutover-runbook.md** — freeze, final verified run, flip, rollback trigger, NAS decommission checklist

## Example Output

> **Cost comparison — 40 TB / 12M files / 500 Mbps link**
> DataSync over the wire: 40,000 GB × $0.0125 = **$500** transfer + $60 S3 PUTs (12M × $0.005/1k). Calendar: 40 × 116 ÷ 400 usable Mbps ≈ **11.6 days** at full rate; ~18 days on the throttled calendar.
> Snowball Edge 210 TB: **$1,800** job fee + ~$80 shipping; ~2–3 weeks end-to-end.
> **Verdict:** 11.6 < 14 break-even days → **DataSync wins on cost ($560 vs ~$1,880) and roughly ties on time. Proceed over the wire.**
> Acceptance gate: final execution `exec-0fa1b2c3d4e5f6789` → VerifyMode=POINT_IN_TIME_CONSISTENT, FilesVerified: 12,041,337, Verification errors: **0** ✅

## AWS Services Used

- **AWS DataSync** — agent-based transfer, throttling, scheduling, filters, built-in verification, task reports
- **Amazon S3** — destination bucket, task-report bucket, storage-class selection at ingest
- **AWS Storage Gateway (S3 File Gateway)** — the hybrid tail: SMB/NFS access to S3 after the NAS dies
- **AWS Snowball Edge** — the offline alternative when the break-even math says the pipe loses
- **Amazon CloudWatch Logs** — per-file TRANSFER logging, 30-day retention
- **AWS IAM + AWS Secrets Manager** — least-privilege bucket access role; SMB credentials never inline

## Well-Architected Alignment

- **Reliability:** incremental `TransferMode=CHANGED` makes every run resumable; `POINT_IN_TIME_CONSISTENT` verification is the cutover acceptance gate, not an afterthought
- **Cost Optimization:** flat $0.0125/GB priced up front; Snowball break-even computed before committing; storage class chosen at ingest so lifecycle policies start from the right tier
- **Operational Excellence:** scripted runbook, task reports to S3 as audit evidence, scheduled executions instead of humans babysitting terminals at 3 a.m.
- **Security:** TLS 1.2+ on the wire via the agent's 443-only egress, bucket role scoped to one prefix, SMB password in Secrets Manager
- **Performance Efficiency:** agent sized to the file count (32 GB vs 64 GB RAM at the 20M-file line), 10 Gbps per-task ceiling respected, throttle lifted only when the office sleeps

## Cost Notes

Worked example (40 TB, 12M files, us-east-1):

- DataSync Basic mode: 40,000 GB × **$0.0125/GB = $500** one-time (Enhanced mode would be $0.015/GB + $0.55/execution—unnecessary here)
- S3 PUT requests: 12M × $0.005/1,000 = **$60**
- CloudWatch TRANSFER logs: ~**$5–15** for the migration window
- S3 Standard at rest: 40,000 GB × $0.023 = **$920/month** until lifecycle rules tier it down
- Snowball Edge alternative: **$1,800** (≤100 TB) or **$3,200** (101–210 TB) per job, 15 days onsite included, $250/day after, import to S3 free—wins whenever network days (TB × 116 ÷ usable Mbps) exceed ~14
- Optional S3 File Gateway tail: $0.01/GB written, **capped at $125/month** per gateway

Total to move 40 TB over the wire: **~$575 one-time**. The NAS replacement it avoids: ~$15,000.

## Troubleshooting

1. **Agent shows OFFLINE after activation** — Cause: outbound TCP 443 to DataSync endpoints blocked, or VM clock skew breaking TLS. → Fix: open 443 egress from the agent subnet (or create a DataSync VPC endpoint) and enable NTP on the hypervisor host.
2. **Execution fails in PREPARING on a big tree** — Cause: file count over 20M on a 32 GB agent, or over the 50M Basic-mode per-execution quota. → Fix: resize the agent to 64 GB RAM; above 50M files, split into per-share tasks with include filters instead of requesting a quota bump.
3. **Final run reports verification errors** — Cause: users wrote to the NAS during the cutover sync, so source and destination legitimately diverge. → Fix: remount the share read-only, rerun the final task, and only accept a report with zero errors.
4. **Office link still saturated despite the throttle** — Cause: `BytesPerSecond` is per task execution; three concurrent share-tasks at 100 Mbps each is 300 Mbps. → Fix: stagger schedules an hour apart or divide the budget (e.g., 4,166,666 bytes/s each).
5. **SMB location creation fails with mount errors** — Cause: wrong domain format or an account without share-level read permission. → Fix: pass `--domain CORP` separately from `--user`, pull the password from Secrets Manager, and test the same credentials from another machine first.

**Bar:** scripts deploy in under 15 minutes; cutover signs off with zero unverified bytes.
