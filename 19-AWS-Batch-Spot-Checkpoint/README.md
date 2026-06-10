# AWS Batch on EC2 Spot with Checkpoint-Resume: Survive the 2-Minute Interruption Notice Without Losing Work

Run multi-hour batch and ML jobs on EC2 Spot through AWS Batch—wire the 2-minute interruption handler, checkpoint to Amazon S3 faster than the notice, and resume the exact job that died—so you keep up to 70% off on-demand instead of restarting a 9-hour run from zero.

## The Problem

EC2 Spot is the cheapest compute AWS sells—commonly 60-70% off on-demand, up to 90% on quiet capacity pools—but AWS reclaims instances with only a **2-minute interruption notice**. Get a 9-hour genomics, rendering, or training job reclaimed at hour 7 and you lose 7 hours of compute AND the dollars behind it, so teams retreat to on-demand and overpay 3x.

Concrete numbers: 9 hours on `c7i.2xlarge` in us-east-1 runs about $0.357/hr on-demand = $3.21/run; Spot typically lands near $0.11-0.13/hr = $1.00-1.17/run. At 500 runs/month that is roughly $1,605 vs ~$530—**but only if interruptions do not force full restarts.** The fix: a handler that checkpoints to S3 faster than the 120-second window closes, plus a `retryStrategy` that resumes the same job from the last checkpoint.

## Who This Is For

- Startup ML, simulation, rendering, ETL, and genomics teams running multi-hour AWS Batch jobs who want Spot pricing without losing work.
- Platform and FinOps owners who need a defensible "use Spot, save 60-70%" decision backed by a tested resume mechanism.
- Engineers with a Dockerized batch workload who want a production-ready compute environment, handler, and checkpoint contract to drop in.

## How to Use

1. Open Claude Code, Kiro CLI, or Amazon Q Developer CLI in the repo holding your batch container.
2. Connect the AWS Documentation MCP server so the assistant verifies current Spot allocation strategies, retry parameters, and pricing—not memory.
3. Paste the System Prompt below and replace [REGION], [JOB_RUNTIME_HOURS], [VCPU], [MEMORY_MIB], [CHECKPOINT_S3_URI], [INSTANCE_FAMILIES].
4. Let it generate the compute environment, job queue, job definition, handler, and cadence math.
5. Review the math and cost table, deploy, then force a real Spot interruption with AWS Fault Injection Service to prove resume.

Prerequisites:
- Required Access: An IAM principal that can create Batch resources, plus `iam:PassRole` for the Batch service role and `ecsInstanceRole`. Trial with `AWSBatchFullAccess`, then tighten. The job role needs `s3:GetObject`/`PutObject`/`ListBucket` on the checkpoint prefix only.
- Recommended Background: The Spot 2-minute notice, Batch job states, and how your workload serializes state.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`), AWS CLI v2, Docker, and an image in Amazon ECR.

Key Parameters: allocationStrategy=SPOT_PRICE_CAPACITY_OPTIMIZED, bidPercentage=100, minvCpus=0/maxvCpus=256, 6+ instance types across 2+ AZs, retryStrategy attempts=5 with ordered evaluateOnExit rules, checkpoint interval=300s, IMDSv2 poll=5s, HttpPutResponseHopLimit=2, attemptDurationSeconds=[runtime x 1.5], S3 Lifecycle expiry 7 days.

Troubleshooting: If an interrupted job goes FAILED with no retry, that is expected when no RETRY rule matches or the job was cancelled/terminated; order the rules RETRY `onStatusReason "Host EC2*"`, RETRY `onExitCode 75`, EXIT `onReason "*"`.

## System Prompt

```
# AWS Batch on EC2 Spot with Checkpoint-Resume — Production Build Request

You are a senior AWS HPC/batch platform engineer. Build a production-ready AWS Batch on EC2 Spot setup that survives a reclaim by checkpointing to Amazon S3 inside the 2-minute interruption notice and resuming the same job from the last checkpoint. Every price, flag, and API field MUST be real and current. NEVER invent a parameter; if a fact is unverifiable, write the safe generic form and flag it. Removal beats invention.

## Target Workload
- Region: [REGION] | Runtime (h): [JOB_RUNTIME_HOURS] | vCPU: [VCPU] | Memory (MiB): [MEMORY_MIB]
- Checkpoints: [CHECKPOINT_S3_URI] | Instance families: [INSTANCE_FAMILIES] (e.g., c7i, c6i, m7i)

## Step 0 — Verify Reality FIRST
1. CONFIRM via the AWS Documentation MCP server (aws_read_documentation): the Spot interruption notice is 2 minutes, delivered via the EventBridge "EC2 Spot Instance Interruption Warning" event and IMDSv2 /latest/meta-data/spot/instance-action (poll every 5 seconds); the Instance Rebalance Recommendation is an earlier best-effort signal.
2. Confirm AWS Batch supports allocationStrategy SPOT_PRICE_CAPACITY_OPTIMIZED and retryStrategy attempts 1-10 with up to 5 evaluateOnExit rules (onStatusReason/onReason/onExitCode, RETRY|EXIT, first match wins); cancelled or terminated jobs are NEVER retried.
3. Pull live on-demand and Spot prices for the primary instance type (aws ec2 describe-spot-price-history); check interruption frequency in the Spot Instance Advisor.

## Step 1 — Checkpoint Cadence Math (show every number)
- The notice is 120s. T_upload = largest checkpoint / S3 throughput. Flush budget = 120s - T_upload - 15s safety; required: T_upload + 15s <= 120s.
- Pick the periodic interval (default 300s); worst-case lost recompute = one interval. If full state cannot upload in 120s, periodic checkpointing is primary and the on-notice flush best effort. Multipart upload above 100 MiB.

## Step 2 — Infrastructure (Terraform preferred; exact fields)
1. MANAGED Spot compute environment: type SPOT, allocationStrategy SPOT_PRICE_CAPACITY_OPTIMIZED, bidPercentage 100, minvCpus 0, desiredvCpus 0, maxvCpus 256, >=6 instance types across >=2 subnets/AZs, an instanceRole, a launch template with HttpPutResponseHopLimit=2 (the container must reach IMDSv2). spotIamFleetRole ONLY for legacy BEST_FIT. Tag Environment/Owner/CostCenter.
2. A priority-1 job queue; optional lower-priority on-demand fallback.
3. A job definition: vcpus [VCPU], memory [MEMORY_MIB], the ECR image, env vars CHECKPOINT_S3_URI and CHECKPOINT_INTERVAL_SECONDS, a job role with s3:GetObject/PutObject/ListBucket on the checkpoint prefix ONLY, attemptDurationSeconds = [JOB_RUNTIME_HOURS]*3600*1.5, retryStrategy attempts 5, evaluateOnExit in order: RETRY onStatusReason "Host EC2*", RETRY onExitCode 75, EXIT onReason "*".
4. An S3 Lifecycle rule expiring the checkpoint prefix after 7 days.

## Step 3 — Interruption Handler (the resume contract)
Generate a handler (shell or Python) in the entrypoint:
- START: if a checkpoint exists in [CHECKPOINT_S3_URI] for this logical job, download and RESUME. Key checkpoints on a stable job id, NOT AWS_BATCH_JOB_ATTEMPT, so attempt 2 finds attempt 1's state.
- PERIODIC: checkpoint to S3 every CHECKPOINT_INTERVAL_SECONDS.
- ON NOTICE: poll IMDSv2 spot/instance-action every 5s (token via PUT /latest/api/token; 404 = none pending); on a present action, flush the latest state and exit 75 so the RETRY rule fires—exit 0 would never retry. A rebalance recommendation triggers one extra checkpoint.
- Log every write/restore with byte size and elapsed seconds to stdout.

## Step 4 — Error Handling (Cause -> Resolution)
Cover: FAILED with no retry (no matching RETRY rule or cancelled/terminated -> fix rule order/exit code); resume from zero (keyed on attempt number -> stable job id); stuck RUNNABLE (no Spot capacity or broken iam:PassRole -> widen types/AZs, verify roles); flush killed mid-upload (shrink interval, multipart); IMDSv2 token PUT hangs in-container (hop limit 1 -> set 2).

## Step 5 — Evidence & Acceptance
Produce, in order: (1) cadence math with concrete seconds; (2) IaC files; (3) handler script; (4) AWS CLI to submit a test job and force a REAL interruption with the AWS FIS action aws:ec2:send-spot-instance-interruptions, confirming AWS_BATCH_JOB_ATTEMPT increments and the new attempt resumes from S3, not zero; (5) a checklist: CloudWatch Logs restore lines, S3 checkpoint object versions, attempt 2 SUCCEEDED; (6) a cost table at live [REGION] prices, per run and per month. The result must be production-ready; a forced interruption must be survivable and verifiable in under 10 minutes.
```

## What You Get

1. A managed EC2 Spot compute environment (Terraform/CloudFormation): `SPOT_PRICE_CAPACITY_OPTIMIZED`, `bidPercentage=100`, `minvCpus=0`/`maxvCpus=256`, 6+ types across 2+ AZs, `HttpPutResponseHopLimit=2`, plus a priority-1 job queue and optional on-demand fallback.
2. A job definition with the ordered Spot-aware `retryStrategy` (attempts=5) and an `attemptDurationSeconds` timeout at 1.5x runtime.
3. The interruption handler: resumes from S3 on start, checkpoints every 300s, polls IMDSv2 every 5s, flushes on notice, exits 75 so Batch retries.
4. The checkpoint contract keyed on a stable job id (never `AWS_BATCH_JOB_ATTEMPT`), with 7-day S3 Lifecycle expiration.
5. AWS CLI to submit a test job and force a real interruption via AWS FIS (`aws:ec2:send-spot-instance-interruptions`).
6. A verification checklist (CloudWatch Logs restore lines, S3 object versions, attempt 2 SUCCEEDED) and a live-price cost table.

## Example Output

Cadence math (us-east-1, c7i.2xlarge, 9h job): state ~1.2 GiB, S3 upload ~40s, flush budget = 120s - 40s - 15s = 65s; interval 300s, worst-case lost work = 5 min. FIS forces a real interruption; attempt 1 flushes and exits 75, attempt 2 restores 1.2 GiB in 38s and SUCCEEDS. Cost: $3.21/run on-demand vs ~$1.08/run Spot (~66% off); 500 runs/month: ~$1,605 vs ~$540 = ~$1,065 saved.

## AWS Services Used

AWS Batch, Amazon EC2 Spot Instances, Amazon S3, Amazon ECR, AWS Fault Injection Service, Amazon EventBridge, Amazon CloudWatch Logs, AWS IAM, IMDSv2, AWS CLI, Terraform (or AWS CloudFormation).

## Well-Architected Alignment

- Cost Optimization: Spot at commonly 60-70% (up to 90%) off on-demand with `SPOT_PRICE_CAPACITY_OPTIMIZED`; the cost table uses live `describe-spot-price-history` prices.
- Reliability: the handler, S3 checkpoints, and ordered `retryStrategy` turn a reclaim into a resumable event; 6+ types across 2+ AZs lower interruption frequency.
- Operational Excellence: emits IaC, a forced-interruption test via AWS FIS, and CloudWatch Logs evidence so resume behavior is observable and repeatable.
- Performance Efficiency: cadence is computed against the 120-second notice and measured upload time, bounding worst-case recompute to one interval.
- Security: the job role is scoped to the checkpoint prefix only; instances enforce IMDSv2 with hop limit 2.

## Cost Notes

- Spot for c7i/c6i/m7i/m6i commonly runs 60-70% below on-demand, up to 90% on quiet pools; the prompt verifies the live, pool-dependent price before quoting savings.
- Worked example (us-east-1): `c7i.2xlarge` ~$0.357/hr on-demand; a 9-hour job is ~$3.21 vs ~$1.00-1.17 on Spot—about $1,000/month saved at 500 runs.
- `minvCpus=0` scales the environment to zero between jobs; S3 checkpoint storage is the only standing cost, kept negligible by the 7-day Lifecycle expiration (~$0.023/GiB-month, S3 Standard).
- The on-demand fallback only runs when Spot capacity is unavailable, so it does not erase the savings.

## Troubleshooting

- Job FAILED with no retry after a reclaim. Cause: no RETRY rule matched, or the job was cancelled/terminated (never retried). Fix: order rules RETRY `onStatusReason "Host EC2*"`, RETRY `onExitCode 75`, EXIT `onReason "*"`, and exit 75 on notice.
- Resume restarts from zero. Cause: the checkpoint key includes `AWS_BATCH_JOB_ATTEMPT`. Fix: key on a stable logical job id so every attempt reads the same S3 prefix.
- Jobs stuck RUNNABLE at desiredvCpus 0. Cause: no Spot capacity or broken `iam:PassRole`. Fix: broaden to 6+ types across 2+ AZs; verify the instance and service roles.
- On-notice checkpoint never writes. Cause: no polling, or the IMDSv2 token PUT dies at the container boundary (hop limit 1). Fix: set `HttpPutResponseHopLimit=2`; a 404 on the poll just means no interruption pending.
- Job killed mid-flush. Cause: state too large for 120 seconds. Fix: lower the periodic interval and use multipart upload—periodic checkpoints carry reliability.
