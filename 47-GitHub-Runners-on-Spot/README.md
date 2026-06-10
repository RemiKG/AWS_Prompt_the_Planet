# Ephemeral GitHub Actions Runners on EC2 Spot: JIT One-Job Clean Rooms That End the Per-Minute CI Tax

Scale-from-zero self-hosted GitHub Actions runners on EC2 Spot with just-in-time registration, one-job-then-terminate lifecycle, and honest Spot-interruption handling—so your CI bill drops 80-90% without inheriting a fleet of stale, poisoned build hosts.

## The Problem

GitHub-hosted Linux runners bill private repos $0.006/minute for 2 vCPU and $0.012/minute for 4 vCPU (the rates effective January 2026). A team burning 50,000 CI minutes a month on 4-vCPU runners pays **$600/month for compute that costs roughly $66 on EC2 Spot**. That is a ~9x markup on commodity vCPUs.

The usual escape—long-lived self-hosted runners—trades the bill for worse problems: runners idle 18 hours a day still cost money, build caches and credentials accumulate on disk across hundreds of jobs, and one `npm install` of a malicious transitive dependency poisons every job that lands on that host afterward. Persistent self-hosted runners are the single most common GitHub Actions security finding.

The fix is ephemeral runners: a webhook fires when a job queues, a Spot instance launches, registers itself with a single-use JIT config, runs **exactly one job**, and terminates. No idle cost. No state carried between jobs. No registration token sitting on a long-lived box. This prompt generates the complete production-ready Terraform for that architecture.

## Who This Is For

- Startups spending $200+/month on GitHub-hosted runner minutes who want the same `runs-on:` ergonomics at Spot prices
- Platform engineers who tried persistent self-hosted runners and got burned by idle cost, disk-full failures, or cross-job contamination
- Teams who need bigger runners (8-16 vCPU builds) where GitHub-hosted pricing ($0.022-$0.042/min) gets painful fastest

Not for: public repositories accepting fork PRs from strangers—the SECURITY.md this prompt generates tells you exactly why, with no hand-waving.

## How to Use

1. Install the AWS Documentation MCP server and configure AWS credentials for the target account (`aws sts get-caller-identity` must succeed).
2. Create a GitHub App in your org: organization permission "Self-hosted runners: Read and write", subscribed to "Workflow jobs" webhook events. Note the App ID and download the private key.
3. Store the private key and a generated webhook secret in AWS Secrets Manager (one secret, two keys).
4. Paste the System Prompt below into Kiro CLI, Claude Code, or Amazon Q Developer CLI. Replace the bracketed placeholders `[GITHUB_ORG]`, `[AWS_REGION]`, `[MAX_FLEET_SIZE]`.
5. Review the generated Terraform, run `terraform plan`, then `terraform apply`.
6. Point the GitHub App webhook URL at the generated API Gateway endpoint, then push the test workflow from VERIFICATION.md.

**Prerequisites**

- **Required Access:** AWS account with permissions for EC2, Auto Scaling, Lambda, API Gateway, Secrets Manager, EventBridge, CloudWatch, IAM; GitHub organization owner (to create the App).
- **Recommended Background:** Basic Terraform, GitHub Actions workflow syntax, familiarity with what a Spot interruption is.
- **Tools Required:** Terraform >= 1.5, AWS CLI v2, and the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) so the assistant can verify API shapes instead of guessing.

**Key Parameters:** Instance pool (c6i.large/c6a.large/m6i.large/m6a.large, swap to xlarge for 4-vCPU parity), max fleet size (20), idle-reaper window (15 min), boot volume (gp3 50 GiB), runner labels (`[self-hosted, linux, x64, spot]`), on-demand fallback label (`on-demand`), webhook Lambda (Python 3.12, 256 MB, 10 s timeout), Spot allocation strategy (price-capacity-optimized).

**Troubleshooting:** If the first job sits queued forever and no instance launches, open the GitHub App's Advanced > Recent Deliveries log. A 401 response means the webhook secret in Secrets Manager does not match the one configured on the App—fix the secret and click Redeliver; this is expected on first wiring and is not a Terraform problem.

## System Prompt

```
# Ephemeral GitHub Actions Runner Fleet on EC2 Spot — Architecture Design Request

## Project Overview
Design and generate production-ready Terraform for a self-hosted GitHub Actions
runner fleet on EC2 Spot in [AWS_REGION] for the GitHub organization [GITHUB_ORG].
Every runner is ephemeral: it registers via a just-in-time (JIT) config, executes
exactly one job, and terminates. Target: replace GitHub-hosted Linux runners
($0.006/min for 2 vCPU, $0.012/min for 4 vCPU on private repos, Jan 2026 rates)
at 80-90% lower compute cost, scaling from zero with no idle runners.

## Verify Reality First — before generating any code
1. Run `aws sts get-caller-identity` and confirm the account and [AWS_REGION].
   Use aws_read_documentation to confirm any AWS API, CLI flag, or GitHub REST
   endpoint you are not 100% certain of. Never invent an API.
2. Check the Spot vCPU quota "All Standard (A, C, D, H, I, M, R, T, Z) Spot
   Instance Requests" via `aws service-quotas get-service-quota`; warn if the
   limit is below 32 vCPUs and include the quota-increase command.
3. Run `aws ec2 describe-spot-price-history --instance-types c6i.large c6a.large
   m6i.large m6a.large --product-descriptions "Linux/UNIX"` and use the REAL
   current prices in COST_MODEL.md — never fabricate a Spot price.
4. Confirm a GitHub App exists with organization permission "Self-hosted
   runners: Read and write" and a "Workflow jobs" webhook subscription, and that
   its private key plus webhook secret are in one Secrets Manager secret. If
   missing, output the exact creation steps and stop.

## Detailed Requirements
### 1. Scale-from-zero control plane
- API Gateway HTTP API -> Lambda (Python 3.12, 256 MB, 10 s timeout) receives
  `workflow_job` webhooks. Validate the `X-Hub-Signature-256` HMAC before any
  other processing; reject with 401 on mismatch. On action `queued` with labels
  matching `[self-hosted, linux, x64, spot]`, increment ASG desired capacity,
  capped at [MAX_FLEET_SIZE] (default 20).
- Idle-reaper Lambda on a 10-minute EventBridge schedule: terminate any fleet
  instance older than 15 minutes that never claimed a job (webhook/runner races
  make orphans inevitable; reap them, do not pretend they cannot happen).

### 2. Ephemeral JIT runners — one job, then terminate
- Auto Scaling group with MixedInstancesPolicy: c6i.large, c6a.large, m6i.large,
  m6a.large across 3 AZs, SpotAllocationStrategy = price-capacity-optimized,
  OnDemandPercentageAboveBaseCapacity = 0 (100% Spot), scale-in protection on
  (runners must self-terminate, never be reaped mid-job).
- User data: fetch the GitHub App key from Secrets Manager, mint an installation
  token, call POST /orgs/[GITHUB_ORG]/actions/runners/generate-jitconfig with a
  unique name, runner_group_id 1, and the labels above; download the latest
  actions/runner release; start with `./run.sh --jitconfig <encoded_jit_config>`.
  On process exit, self-terminate via `aws autoscaling
  terminate-instance-in-auto-scaling-group --should-decrement-desired-capacity`.
  JIT runners are removed by GitHub after one job — never use persistent
  `./config.sh --token` registration.
- IMDSv2 required with HttpPutResponseHopLimit 1, gp3 50 GiB boot volume,
  Amazon Linux 2023 base, no SSH key pair.

### 3. Spot interruption mid-job — handle it honestly
- systemd watcher polls http://169.254.169.254/latest/meta-data/spot/instance-action
  every 5 s; on the 2-minute interruption warning, emit a CloudWatch metric
  (Namespace CIRunners, Metric SpotInterruption) and tag the instance so the
  failure is attributable.
- EventBridge rule on detail-type "EC2 Spot Instance Interruption Warning" ->
  Lambda that calls POST /repos/{owner}/{repo}/actions/runs/{run_id}/rerun-failed-jobs
  once the run completes as failed.
- State plainly in code comments: an interrupted job FAILS and re-runs from the
  start — there is no checkpointing. Jobs labeled `on-demand` (deploys,
  releases, anything non-idempotent) route to a separate 100% On-Demand ASG.

### 4. Security isolation — state it honestly
- Private repositories only. Generate SECURITY.md stating: ephemeral is NOT a
  sandbox — job code still runs with the instance profile, network egress, and
  IMDS access of the host. One job, one instance limits persistence, not blast
  radius during the job.
- Instance profile: exactly secretsmanager:GetSecretValue on the one secret,
  autoscaling:TerminateInstanceInAutoScalingGroup scoped by ResourceTag, and
  cloudwatch:PutMetricData — nothing else. Deployment credentials come from
  GitHub OIDC role assumption inside workflows, never from the instance role.
- Security group: zero inbound rules; egress TCP 443 only.

### 5. Cost model
- COST_MODEL.md: per-minute math vs GitHub-hosted at 10,000 and 50,000
  min/month, including EBS gp3, the ~90 s boot-and-register overhead per job,
  Lambda, API Gateway, and Secrets Manager ($0.40/month).

## Deliverables Requested
1. Terraform, max 6 files: ASG + launch template, webhook stack, IAM, security
   group, variables with the defaults above (Terraform preferred, no modules
   from registries)
2. userdata.sh.tpl, interruption-watcher.service, scale_up.py, idle_reaper.py,
   rerun_on_interruption.py
3. COST_MODEL.md and SECURITY.md as specified
4. VERIFICATION.md: a test workflow using `runs-on: [self-hosted, linux, x64,
   spot]`, with acceptance criteria — instance running within 90 s of queue,
   job green, instance terminated within 2 min of job end, fleet at zero after
   15 idle minutes, CloudWatch alarm on GroupDesiredCapacity >= [MAX_FLEET_SIZE]

Align with the Well-Architected Cost Optimization, Security, and Reliability
pillars. Output the files directly without any preamble. Measurable bar: first
green job on a Spot runner within 30 minutes of `terraform apply`.
```

## What You Get

1. **main.tf** — ASG with MixedInstancesPolicy (4 instance types x 3 AZs, price-capacity-optimized, 100% Spot), launch template with IMDSv2 hop-limit 1, plus the optional On-Demand ASG for `on-demand`-labeled jobs
2. **webhook.tf** — API Gateway HTTP API, scale-up Lambda, idle-reaper Lambda on a 10-minute schedule, EventBridge interruption rule
3. **iam.tf / sg.tf** — least-privilege instance profile (3 actions total) and a zero-inbound security group
4. **userdata.sh.tpl** — Secrets Manager fetch, GitHub App token mint, `generate-jitconfig` call, `run.sh --jitconfig`, self-terminate-with-decrement on exit
5. **interruption-watcher.service** — 5-second IMDS poller emitting the SpotInterruption CloudWatch metric
6. **scale_up.py, idle_reaper.py, rerun_on_interruption.py** — Lambda sources with HMAC validation
7. **COST_MODEL.md** — the per-minute math against real `describe-spot-price-history` output
8. **SECURITY.md** — the honest isolation statement, private-repos-only policy, and fork-PR risk explanation
9. **VERIFICATION.md** — test workflow plus five pass/fail acceptance checks

## Example Output

```markdown
# COST_MODEL.md (excerpt)
| Scenario: 50,000 min/mo, 4 vCPU      | GitHub-hosted | Spot fleet |
|---------------------------------------|---------------|------------|
| Compute (833 hr)                      | $600.00       | $56.60     |
| Boot overhead (~90 s x 2,400 jobs)    | included      | $4.10      |
| EBS gp3 50 GiB (attached hours only)  | included      | $4.80      |
| API GW + Lambda + Secrets Manager     | n/a           | $0.55      |
| **Total**                             | **$600.00**   | **$66.05** |
Savings: 89.0%. Break-even vs setup effort: ~month one.
# VERIFICATION.md check 3: instance terminated within 2 min of job end — PASS
# (i-0a1b2c3d state shutting-down 47 s after workflow_job.completed)
```

## AWS Services Used

Amazon EC2 (Spot), Amazon EC2 Auto Scaling, AWS Lambda, Amazon API Gateway, AWS Secrets Manager, Amazon EventBridge, Amazon CloudWatch, AWS IAM, Amazon EBS

## Well-Architected Alignment

- **Cost Optimization:** 100% Spot via price-capacity-optimized allocation across 12 capacity pools; scale-from-zero so idle cost is $0; per-second EC2 billing vs GitHub's per-minute rounding; COST_MODEL.md built from live Spot prices, not estimates.
- **Security:** Single-use JIT registration (no reusable runner tokens at rest), 3-action instance profile, IMDSv2 with hop limit 1, zero-inbound security group, deployment credentials via GitHub OIDC instead of instance roles, and an explicit written boundary on what ephemeral does NOT protect against.
- **Reliability:** Interruption warnings surfaced as CloudWatch metrics, automated `rerun-failed-jobs` recovery, non-idempotent jobs routed to On-Demand, idle reaper for orphaned capacity, fleet-size alarm.
- **Operational Excellence:** Everything is Terraform; VERIFICATION.md turns "it works" into five checkable assertions; failure modes are documented before they happen.

## Cost Notes

- GitHub-hosted Linux, private repos (Jan 2026 rates): $0.006/min (2 vCPU), $0.012/min (4 vCPU), $0.022/min (8 vCPU), $0.042/min (16 vCPU). 50,000 4-vCPU minutes = **$600/month**.
- c6i.xlarge (4 vCPU) On-Demand in us-east-1: $0.17/hr; Spot typically 55-70% below that. 833 runtime hours ~ **$57-75/month** on Spot — even at full On-Demand it is ~$142, still 76% off.
- Fixed overhead: Secrets Manager $0.40/mo, API Gateway HTTP API $1.00 per million requests, Lambda within free tier at CI scale, gp3 at $0.08/GB-month billed only while volumes are attached.
- Honest tax: ~90 s boot-and-register per job (cut to ~35 s with a pre-baked AMI), and interrupted jobs re-run from zero. Budget ~5% re-run overhead; most Spot pools interrupt under 5% of instances per the Spot Advisor.

## Troubleshooting

1. **Job queued, no instance launches.** Cause: webhook HMAC mismatch (401 in the App's Recent Deliveries) or `runs-on` labels not matching the JIT config labels exactly. Fix: sync the webhook secret in Secrets Manager, ensure workflows use all four labels `[self-hosted, linux, x64, spot]`, then Redeliver.
2. **Instance launches but never claims the job.** Cause: runner group policy blocks the repo, or user data failed before registration (check `/var/log/cloud-init-output.log`). Fix: allow the repo in the default runner group (runner_group_id 1) and verify Secrets Manager access from the instance profile.
3. **Job fails with "The self-hosted runner lost communication with the server."** Cause: Spot interruption mid-job. Fix: confirm the SpotInterruption metric fired, let `rerun_on_interruption.py` re-run it; chronic offender pools mean you should add more instance types or move that job to the `on-demand` label.
4. **ASG desired capacity climbs but instances die instantly.** Cause: Spot capacity not available in chosen pools (`capacity-not-available` in activity history). Fix: keep all 4 types across 3 AZs, verify the Spot vCPU quota, and trust price-capacity-optimized rather than pinning one type.
5. **Fleet never returns to zero.** Cause: orphan runners from webhook races, or `completed` events arriving for jobs other runners took. Fix: the 15-minute idle reaper handles this by design — if instances outlive it, check the reaper Lambda's CloudWatch Logs for missing `autoscaling:TerminateInstanceInAutoScalingGroup` permission.
