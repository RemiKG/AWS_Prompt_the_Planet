# AWS FIS Game-Day Chaos Kit: Five Guardrailed Fault-Injection Experiments with CloudWatch Abort Alarms

Five escalating AWS Fault Injection Service experiments—each with a written hypothesis, a quantified blast radius, automatic CloudWatch abort alarms, and an operator runbook—so multi-AZ failover gets proven on a calendar invite, not discovered during an outage.

## The Problem

Most teams pay for resilience they have never tested. You run a 3-instance Auto Scaling group across 3 AZs, an ECS service with `desiredCount: 4`, an ALB with 30-second health checks—and not one of those mechanisms has ever fired under real failure. Dashboards stay green right up until the first genuine AZ event, when you discover the ASG takes 11 minutes to replace capacity because the launch template references a deprecated AMI, or that your app's connection pool pins dead endpoints for 5 minutes because nobody ever set a client timeout below the OS default of 127 seconds.

The common cop-out is "alert-proving": manually tripping a CloudWatch alarm to confirm the PagerDuty hook works. That tests notification plumbing, not recovery. Real chaos engineering injects the actual fault—terminated instances, CPU saturation, 200 ms of injected latency, killed ECS tasks, a disrupted AZ—and measures whether the system heals within a stated bound. AWS FIS makes this safe and cheap ($0.10 per action-minute; a full five-experiment game day costs under $5), but only if every experiment ships with stop conditions that abort automatically when customer-facing metrics degrade. Teams skip game days because designing safe experiments feels like a week of work. This prompt compresses it to one session.

## Who This Is For

- Startups (2–20 engineers) running EC2 Auto Scaling groups or ECS services behind an ALB who have never run a game day
- Platform/SRE engineers who need a defensible first chaos program with auditable guardrails before touching production
- Teams whose compliance or enterprise customers ask "how do you validate failover?" and need evidence, not architecture diagrams

## How to Use

1. Open Kiro CLI, Claude Code, or Amazon Q Developer CLI in a terminal with AWS credentials for your **staging** account first (`aws sts get-caller-identity` to confirm).
2. Copy the entire System Prompt below into the assistant. Replace the bracketed placeholders `[WORKLOAD_NAME]`, `[REGION]`, and `[COMPUTE_TYPE]` before sending.
3. Let the assistant run read-only discovery and review `discovery-report.md`—confirm it found YOUR resources, not hypothetical ones.
4. Apply the IAM role and guardrail alarms it generates, then run `verify.sh`. Do not start any experiment until every check prints PASS.
5. Schedule the 2-hour game day, run experiments E1→E5 in order, and file the retro notes from each runbook.

**Prerequisites**

- **Required Access:** An AWS account (staging strongly recommended for the first run); IAM permissions to create roles (`iam:CreateRole`, `iam:AttachRolePolicy`, `iam:PassRole`), FIS experiment templates (`fis:CreateExperimentTemplate`, `fis:StartExperiment`), and CloudWatch alarms (`cloudwatch:PutMetricAlarm`).
- **Recommended Background:** CloudWatch alarms and ALB target-group metrics; Auto Scaling group or ECS service fundamentals; basic incident-response process.
- **Tools Required:** Kiro CLI or Claude Code with the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) for action/Region verification; AWS CLI v2 configured; SSM Agent running on EC2 targets with the `AmazonSSMManagedInstanceCore` instance-profile policy (required for the stress actions).

**Key Parameters:** CPU stress (80% load / 300 s), injected latency (200 ms / 300 s), target selection (COUNT(1) for terminations, PERCENT(30) for stress), AZ disruption window (PT10M), abort thresholds (5xx > 50/min, p99 > 2 s, UnHealthyHostCount >= 2), FIS log retention (90 d), game-day budget cap ($10).

**Troubleshooting:** If `verify.sh` reports a stop-condition alarm stuck in INSUFFICIENT_DATA, that's expected on freshly created alarms—FIS stop conditions only trigger on the ALARM state, so an alarm with no datapoints protects nothing. Drive a few minutes of traffic (or temporarily set `--treat-missing-data notBreaching`) until every guardrail alarm reads OK before injecting anything.

## System Prompt

```
# AWS FIS Game-Day Chaos Program — Design Request

## Project Overview
Design a production-ready game-day chaos program for my workload [WORKLOAD_NAME] in [REGION], running on [COMPUTE_TYPE: EC2 Auto Scaling group | ECS on EC2 | ECS on Fargate] behind an Application Load Balancer. I want true fault injection that tests recovery hypotheses — NOT alert-proving exercises that merely confirm alarms page someone. Every experiment must ship with a written hypothesis, a quantified expected blast radius, CloudWatch alarm stop conditions that abort the experiment automatically, and an operator runbook.

## Reality Check — Do This FIRST, Before Generating Anything
1. Discover my actual resources with read-only calls only: aws ec2 describe-instances, aws autoscaling describe-auto-scaling-groups, aws ecs list-services, aws elbv2 describe-load-balancers, aws elbv2 describe-target-groups, aws cloudwatch describe-alarms. Write findings to discovery-report.md. NEVER target a resource you have not confirmed exists; never invent ARNs — every ARN in a template must come from discovery output.
2. Verify with the AWS Documentation MCP server (aws_read_documentation) that AWS FIS supports every action you propose in [REGION]. If an action is unsupported there, substitute the closest supported action and say so explicitly.
3. Confirm EC2 targets are managed by Systems Manager (aws ssm describe-instance-information) — the stress actions run via SSM and fail silently-looking otherwise.
4. Confirm every alarm referenced in stopConditions exists AND is in OK state. FIS only aborts on ALARM state; an alarm in INSUFFICIENT_DATA protects nothing.
5. If any prerequisite is missing, generate it — do not assume it exists. If a discovery call returns AccessDenied, name the exact missing IAM action and stop.

## Detailed Requirements

### 1. Five Starter Experiments (escalating order)
For each, produce: hypothesis ("We believe X; we will know within Y minutes via metric Z"), a complete FIS experiment template JSON, expected blast radius as an exact resource count and percent of capacity, named abort alarms, max duration, and rollback behavior.
- E1 — Single-instance loss: aws:ec2:terminate-instances, selectionMode COUNT(1), filtered to tag ChaosReady=true inside the ASG. Blast radius: 1 of N instances (<=33% of a 3-instance fleet, one AZ). Hypothesis: ASG restores capacity in under 5 minutes with zero customer-facing 5xx.
- E2 — CPU saturation: aws:ssm:send-command with document AWSFIS-Run-CPU-Stress (DurationSeconds 300, 80% load), selectionMode PERCENT(30). Hypothesis: target-tracking scaling adds capacity before p99 TargetResponseTime breaches 2 s.
- E3 — Dependency latency: aws:ssm:send-command with AWSFIS-Run-Network-Latency injecting 200 ms for 300 s on PERCENT(30) of instances. Hypothesis: 1 s client timeouts plus retries absorb it; 5xx rate stays under 1%.
- E4 — Task churn: aws:ecs:stop-task, selectionMode COUNT(1) per service (skip if no ECS). Hypothesis: the service scheduler restores desiredCount within 120 s and the ALB drains connections with zero dropped requests.
- E5 — AZ disruption (game-day finale): aws:network:disrupt-connectivity with scope "all" against every discovered subnet in ONE availability zone, duration PT10M. Blast radius: all resources in that AZ. Hypothesis: multi-AZ failover completes with under 60 s of elevated errors and no manual action.

### 2. Guardrails (non-negotiable)
- Every template carries at least TWO stopConditions with source aws:cloudwatch:alarm: one workload alarm (HTTPCode_Target_5XX_Count > 50 in 60 s) and one master COMPOSITE alarm OR-ing 5xx burst, p99 TargetResponseTime > 2 s, and UnHealthyHostCount >= 2. Source "none" is forbidden outside a sandbox account — state this in the runbooks.
- One dedicated FIS execution role trusted by fis.amazonaws.com, attaching ONLY the AWS managed policies needed for the chosen actions (AWSFaultInjectionSimulatorEC2Access, AWSFaultInjectionSimulatorECSAccess, AWSFaultInjectionSimulatorNetworkAccess, AWSFaultInjectionSimulatorSSMAccess). Never AdministratorAccess.
- Targets resolve ONLY via tag ChaosReady=true. Tag every template with Environment and Owner.
- Enable experiment logging to CloudWatch Logs group /fis/game-day with 90-day retention.

### 3. Runbooks and Agenda
One runbook per experiment: pre-checks, exact start command (aws fis start-experiment --experiment-template-id ...), what healthy looks like during injection, manual abort (aws fis stop-experiment --id ...), post-checks, and a 5-question retro (Did the hypothesis hold? Time to detect? Time to recover? Surprises? Follow-up actions?). Plus game-day-agenda.md: a 2-hour schedule — 15-min brief, E1–E5 each with a 10-minute observation window, 30-min retro — and a rule that any FAILED hypothesis halts the remaining experiments.

### 4. Cost Discipline
FIS bills $0.10 per action-minute. Show the action-minute math per experiment and cap the full game day under $10. Note the recurring cost of the guardrail alarms.

## Deliverables Requested
1. discovery-report.md  2. fis-templates/e1-instance-loss.json … e5-az-disrupt.json (5 files)  3. iam/fis-execution-role.json + trust policy  4. alarms/guardrail-alarms.sh (CLI, idempotent)  5. runbooks/E1.md … E5.md  6. game-day-agenda.md  7. verify.sh

## Verification & Acceptance
verify.sh must prove readiness BEFORE any injection: all 5 templates exist (aws fis list-experiment-templates), every stop-condition alarm is in OK state, every target query resolves to >0 tagged resources, and the execution role's policies are attached. Print PASS/FAIL per check and exit non-zero on any FAIL. Do not start an experiment until all checks PASS. Output all files as fenced blocks with paths, no preamble. Target: an operator goes from pasting this prompt to a passing verify.sh in under 30 minutes.
```

## What You Get

1. **discovery-report.md** — inventory of the ASG/ECS services, ALB target groups, subnets per AZ, and existing alarms the experiments will reference
2. **fis-templates/** — 5 ready-to-create FIS experiment template JSON files (e1-instance-loss → e5-az-disrupt), each with tag-scoped targets and dual stop conditions
3. **iam/fis-execution-role.json** — least-privilege execution role with the `fis.amazonaws.com` trust policy and only the four needed AWS managed policies
4. **alarms/guardrail-alarms.sh** — idempotent CLI script creating the 5xx burst, p99 latency, and unhealthy-host alarms plus the master composite abort alarm
5. **runbooks/E1.md–E5.md** — operator runbooks: pre-checks, start/abort commands, healthy-vs-abort signals, post-checks, 5-question retro
6. **game-day-agenda.md** — the 2-hour schedule with a halt-on-failed-hypothesis rule
7. **verify.sh** — PASS/FAIL readiness gate that blocks injection until templates, alarms, targets, and the role all check out

## Example Output

> **E1 — Single-Instance Loss** · Hypothesis: terminating 1 of 3 instances in `web-asg` restores capacity in <5 min with zero 5xx. Blast radius: 1 instance (33% of capacity, AZ us-east-1a only). Abort: `alb-5xx-burst` (>50 in 60 s) or `gameday-master-abort` enters ALARM → FIS stops the experiment automatically.
>
> ```
> $ ./verify.sh
> [PASS] 5/5 experiment templates exist
> [PASS] 4/4 stop-condition alarms in OK state
> [PASS] E1 targets resolve: 3 instances tagged ChaosReady=true
> [PASS] fis-gameday-role: 4 managed policies attached
> READY — estimated cost this game day: 26 action-minutes ≈ $2.60
> ```

## AWS Services Used

AWS Fault Injection Service (FIS) · Amazon CloudWatch (alarms, composite alarms) · AWS Systems Manager (Run Command, AWSFIS-Run-* documents) · Amazon EC2 Auto Scaling · Amazon ECS · Elastic Load Balancing (ALB) · AWS IAM · Amazon CloudWatch Logs

## Well-Architected Alignment

- **Reliability** — directly implements REL12 ("Test reliability"): fault-injection testing of recovery procedures with defined hypotheses, instead of assuming Multi-AZ works
- **Operational Excellence** — game days are a named OE best practice; every experiment produces a runbook, a retro, and follow-up actions
- **Security** — dedicated least-privilege FIS execution role, tag-scoped targeting (`ChaosReady=true`), no wildcard resource access, 90-day audit logs in CloudWatch Logs
- **Cost Optimization** — per-experiment action-minute math, a $10 game-day cap, and guardrails that abort spend the moment customers are impacted
- **Performance Efficiency** — E2/E3 validate that scaling policies and timeout budgets hold under load and latency, producing tuning evidence

## Cost Notes

- **AWS FIS:** $0.10 per action-minute. Typical game day: E1 ≈ 1 min ($0.10), E2 ≈ 5 min ($0.50), E3 ≈ 5 min ($0.50), E4 ≈ 1 min ($0.10), E5 = 10 min ($1.00) → **≈ $2.20–$5.00 total** including re-runs
- **CloudWatch:** ~4 standard alarms at $0.10/month + 1 composite at $0.50/month ≈ **$0.90/month** recurring
- **CloudWatch Logs:** FIS experiment logs are kilobytes; ingestion at $0.50/GB rounds to ~$0
- **Replacement capacity:** ASG/ECS replacements are transient minutes of normal On-Demand pricing—budget pennies
- Run quarterly: a full year of game days costs **under $25**, versus one untested failover failing in production.

## Troubleshooting

1. **Experiment ends instantly with state `stopped` and reason "Stop condition triggered"** — Cause: a guardrail alarm was already in ALARM (or flapping) at start. → Fix: run `aws cloudwatch describe-alarms --state-value ALARM` first; `verify.sh` enforces this gate—don't skip it.
2. **"Unable to assume role" when starting an experiment** — Cause: the execution role's trust policy doesn't list `fis.amazonaws.com`, or your caller lacks `iam:PassRole` on it. → Fix: re-apply `iam/fis-execution-role.json` and grant `iam:PassRole` scoped to that role ARN.
3. **E2/E3 stress actions report "completed" but nothing happened** — Cause: target instances aren't SSM-managed (agent stopped or instance profile missing `AmazonSSMManagedInstanceCore`). → Fix: check `aws ssm describe-instance-information`; attach the policy and restart the agent.
4. **"Target resolution returned an empty set"** — Cause: no resources carry `ChaosReady=true`, or PERCENT(30) of a tiny fleet rounded to zero targets. → Fix: tag the fleet explicitly and use COUNT(1) for fleets under 4 instances.
5. **E1 terminated an instance that never came back** — Cause: the target wasn't in an Auto Scaling group—`aws:ec2:terminate-instances` is permanent for standalone instances. → Fix: restrict E1's target filter to instances with the `aws:autoscaling:groupName` tag, or use `aws:ec2:stop-instances` for your first run.

---
**License:** Free to use, adapt, and redistribute.
