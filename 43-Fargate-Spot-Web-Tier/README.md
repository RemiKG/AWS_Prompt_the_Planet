# ECS Fargate Spot for Web Tiers: Capacity-Provider Cutover and 2-Minute SIGTERM Drain Without Dropped Requests

Keeps a guaranteed on-demand floor, sends overflow tasks to Fargate Spot at up to 70% off, and tunes the entire 2-minute drain chain—so Spot reclaims cost you tasks, not customers.

## The Problem

A stateless web tier of ten 1 vCPU / 2 GB Fargate tasks costs **$360.40/month** on-demand in us-east-1 ($0.04048/vCPU-hr + $0.004445/GB-hr). AWS prices Fargate Spot at **up to 70% below** the regular Fargate rate for the exact same tasks — but most teams leave that money on the table for one of two reasons:

1. **They never adopt Spot** because "the web tier can't tolerate interruptions" — false for stateless services behind an ALB, if the drain is engineered.
2. **They flip 100% to Spot** and eat a 5xx storm the first time AWS reclaims capacity. The math fails silently: a reclaimed Fargate Spot task gets a SIGTERM and is SIGKILLed at most **120 seconds** later, while the ALB target group's default `deregistration_delay.timeout_seconds` is **300 seconds**. The load balancer is still draining connections to a container that no longer exists. Worse, Fargate **never backfills reclaimed Spot capacity with on-demand** — a 100% Spot service can sit below desired count for as long as the capacity crunch lasts, and a single-task Spot service goes to zero.

The fix is a capacity-provider strategy with an on-demand `base`, weighted Spot overflow, and three numbers aligned: `stopTimeout=120`, deregistration delay 90, and an application SIGTERM handler that finishes in-flight requests. This prompt generates that cutover for an existing service, plus the telemetry to measure your real interruption rate.

## Who This Is For

- Startups running stateless APIs, SSR frontends, or web apps on ECS Fargate behind an Application Load Balancer who want the Spot discount with production-grade reliability.
- Teams already burned by an all-Spot configuration who need a defensible base/weight split and a drain that survives reclaims.
- Platform engineers standardizing a capacity-provider strategy across many services and deciding which workloads must stay off Spot.

This is **not** for batch/queue workers (use AWS Batch or a separate worker service pattern), stateful services, or long-lived WebSocket gateways without client reconnect logic.

## How to Use

1. Copy the System Prompt below into Kiro CLI or Claude Code — or save it as `.kiro/steering/fargate-spot-cutover.md` to reuse across services.
2. Replace the bracketed placeholders: `[CLUSTER_NAME]`, `[SERVICE_NAME]`, `[TARGET_GROUP_ARN]`, `[RUNTIME]`, `[AWS_REGION]`.
3. Let the assistant run the read-only verification calls (`describe-services`, `describe-task-definition`, `describe-target-group-attributes`) before it generates anything. If it reports a Windows task definition, stop — Fargate Spot does not support Windows containers (Linux x86_64 and ARM64/Graviton are both supported on platform version 1.4.0+).
4. Review the generated files, then `terraform plan && terraform apply` (or run the shell script). The change applies as one forced deployment with zero downtime.
5. Run the `verify-drain.md` runbook: stop one Spot-placed task and confirm ALB 5xx stays at zero through the drain. Re-check your EventBridge interruption log after one week to validate the base/weight ratio.

**Prerequisites**

- **Required Access:** IAM permissions for `ecs:Describe*`, `ecs:PutClusterCapacityProviders`, `ecs:UpdateService`, `ecs:RegisterTaskDefinition`, `ecs:StopTask`, `elasticloadbalancing:ModifyTargetGroupAttributes`, `events:PutRule`, `events:PutTargets`, `sns:CreateTopic`, `cloudwatch:PutMetricAlarm`. An existing ECS Fargate service behind an ALB.
- **Recommended Background:** ECS services and target groups; the AWS Containers blog post "Graceful shutdowns with ECS."
- **Tools Required:** Kiro CLI or Claude Code (any AI assistant works), AWS CLI v2, Terraform >= 1.5 (optional — plain CLI output available), AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) for flag verification.

**Key Parameters:** on-demand base (2), FARGATE:FARGATE_SPOT weight (1:3 → 25/75 overflow split), stopTimeout (120s — Fargate max, default 30s), deregistration_delay.timeout_seconds (90 — ALB default 300), health check interval/healthy/unhealthy (10s/2/2), healthCheckGracePeriodSeconds (60), interruption-log retention (30d).

**Troubleshooting:** If `update-service` returns `InvalidParameterException` saying a capacity provider is not associated with the cluster, that's expected on first run — `FARGATE` and `FARGATE_SPOT` exist account-wide but must be attached per cluster with `put-cluster-capacity-providers`, and that call **replaces** the cluster's entire provider list, so pass every provider the cluster needs in one call.

## System Prompt

```
# ECS Fargate Spot Cutover Request: Stateless Web Tier

Act as a senior AWS infrastructure engineer. Convert my existing ECS Fargate web service to a mixed on-demand + Fargate Spot capacity-provider strategy with a graceful two-minute SIGTERM drain. The result must be production-ready: zero dropped requests during Spot reclaims.

My context: cluster [CLUSTER_NAME], service [SERVICE_NAME], target group [TARGET_GROUP_ARN], app runtime [RUNTIME: node|python|go], region [AWS_REGION].

## Verify Reality First (before generating anything)
1. Confirm the cluster, service, task definition, and target group exist: run `aws ecs describe-services`, `aws ecs describe-task-definition`, and `aws elbv2 describe-target-group-attributes`. Never invent ARNs — if a placeholder is missing, ask me.
2. Confirm the task definition is Linux (x86_64 or ARM64/Graviton — both run on FARGATE_SPOT at platform version 1.4.0+). Fargate Spot does NOT support Windows containers; if the task targets Windows, stop and report it instead of generating a broken strategy.
3. Confirm platform version is 1.4.0 or LATEST (1.4.0+ is required for ARM64/Graviton on FARGATE_SPOT), and inspect the container ENTRYPOINT: a shell-form entrypoint swallows SIGTERM (PID 1 problem) — flag it and prescribe the exec-form fix.
4. Cross-check every CLI flag, Terraform resource type, and target-group attribute against current docs with the aws_read_documentation MCP tool before emitting it.

## Detailed Requirements

### 1. Capacity-Provider Strategy
- Associate FARGATE and FARGATE_SPOT with the cluster via `aws ecs put-cluster-capacity-providers`. This call REPLACES the provider list — include every provider the cluster already uses.
- Service strategy: `capacityProvider=FARGATE,base=2,weight=1` plus `capacityProvider=FARGATE_SPOT,weight=3`: two guaranteed on-demand tasks as the availability floor, then overflow split 25/75. Only one provider may set `base`; always set weights explicitly because the CLI/API default weight is 0.
- Apply with `aws ecs update-service --capacity-provider-strategy ... --force-new-deployment` (a forced new deployment is required when switching from a launch type).
- Comment in the code: Fargate never backfills reclaimed Spot with on-demand — `base` is the only availability floor.

### 2. The 120-Second Drain Budget
- `stopTimeout: 120` in the container definition (Fargate maximum; the 30-second default is too short).
- Target group attribute `deregistration_delay.timeout_seconds=90` — the 300-second default overruns the reclaim window; 90 leaves a 30-second margin before SIGKILL.
- SIGTERM handler for [RUNTIME]: stop accepting new connections, finish in-flight requests, then exit 0. Exec-form ENTRYPOINT so the signal reaches the process.
- ALB health check interval 10s, healthyThreshold 2, unhealthyThreshold 2; `healthCheckGracePeriodSeconds=60` on the service.

### 3. Interruption Observability
- EventBridge rule: source `aws.ecs`, detail-type `ECS Task State Change`, `"stopCode": ["SpotInterruption"]` → SNS topic + CloudWatch Logs group (30-day retention) so I can measure reclaims per week.
- CloudWatch alarms: ALB `HTTPCode_Target_5XX_Count` > 5 over 5 minutes, and service `RunningTaskCount` below base for 5 minutes.

### 4. Cost Target
- At least 40% reduction in this service's Fargate compute line. Basis: Fargate Spot is priced up to 70% below on-demand Fargate ($0.04048/vCPU-hr + $0.004445/GB-hr in us-east-1, Linux/x86); Spot rates float gradually with long-term supply and demand, so pull current rates from the Fargate pricing page before estimating.

### 5. Spot Exclusions (enforce these)
Refuse to place on FARGATE_SPOT: Windows containers (unsupported on Fargate Spot — Linux only), WebSocket/SSE connections longer than 2 minutes without client reconnect, sticky-session-dependent flows, singleton (desired-count 1) services, anything writing state to ephemeral storage, and scheduled jobs or queue consumers — point those at a separate FARGATE-only service and say why.

## Deliverables Requested
1. `spot-cutover.tf` (Terraform preferred; offer `spot-cutover.sh` as the CLI alternative): cluster capacity providers, service strategy, target-group attribute.
2. Task-definition JSON diff adding `stopTimeout`.
3. SIGTERM handler reference implementation for my runtime, with the exec-form ENTRYPOINT line.
4. `spot-observability.tf`: EventBridge rule, SNS topic, both alarms.
5. `verify-drain.md`: an evidence runbook that stops one Spot-placed task with `aws ecs stop-task` and proves zero 5xx via `aws cloudwatch get-metric-statistics` on the ALB — exact commands plus the pass/fail bar.
6. `cost-estimate.md`: before/after monthly table for my task size and count.

Align with the Well-Architected Cost Optimization and Reliability pillars. Output the files only — no preamble. The full cutover must be appliable in under 15 minutes with one forced deployment and zero downtime.
```

## What You Get

1. **`spot-cutover.tf`** — `aws_ecs_cluster_capacity_providers` association, the service's two `capacity_provider_strategy` blocks (base 2 / weights 1:3), and the `aws_lb_target_group` deregistration-delay attribute. A `spot-cutover.sh` CLI variant if you don't run Terraform.
2. **Task-definition diff** — adds `"stopTimeout": 120` to the web container definition.
3. **SIGTERM handler snippet** — runtime-specific (Node/Python/Go): stop the listener, drain in-flight requests, exit 0; plus the exec-form `ENTRYPOINT` correction.
4. **`spot-observability.tf`** — EventBridge rule matching `stopCode: SpotInterruption`, SNS notification topic, CloudWatch Logs interruption ledger (30-day retention), and two alarms (ALB 5xx, RunningTaskCount < base).
5. **`verify-drain.md`** — a runbook that simulates a reclaim with `aws ecs stop-task` and proves zero 5xx through the drain window, with explicit pass/fail criteria.
6. **`cost-estimate.md`** — before/after monthly cost table with the Spot-rate caveat documented.

## Example Output

```hcl
resource "aws_ecs_cluster_capacity_providers" "web" {
  cluster_name       = "web-prod"
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
}

# Service: 2 guaranteed on-demand tasks, then overflow split 25/75.
# NOTE: Fargate never backfills reclaimed Spot with on-demand — base is the floor.
capacity_provider_strategy {
  capacity_provider = "FARGATE"
  base              = 2
  weight            = 1
}
capacity_provider_strategy {
  capacity_provider = "FARGATE_SPOT"
  weight            = 3
}
```

> verify-drain.md, step 3: `aws ecs stop-task --cluster web-prod --task <spot-task-arn> --reason "drain drill"` — PASS if `HTTPCode_Target_5XX_Count` sum = 0 over the next 5 minutes and the replacement task reaches RUNNING within 90 seconds.

## AWS Services Used

Amazon ECS · AWS Fargate (FARGATE and FARGATE_SPOT capacity providers) · Elastic Load Balancing (Application Load Balancer) · Amazon EventBridge · Amazon CloudWatch · Amazon SNS · AWS IAM

## Well-Architected Alignment

- **Cost Optimization:** Spot for fault-tolerant stateless capacity, weighted overflow tuned to your interruption tolerance, and a before/after cost estimate as a required deliverable — the remaining on-demand base stays eligible for Compute Savings Plans (Spot tasks are not).
- **Reliability:** guaranteed on-demand `base` floor, multi-AZ task placement, explicit handling of the no-backfill behavior, `RunningTaskCount` alarm, and a tested 120-second drain chain.
- **Operational Excellence:** interruption-rate telemetry via EventBridge before you trust the ratio, a repeatable drain drill (`verify-drain.md`), and one-command rollback (reapply with weight 0 on FARGATE_SPOT and force a new deployment).
- **Security:** least-privilege IAM action list documented in Prerequisites; no new public surface area is created.

## Cost Notes

Basis stated by AWS: Fargate Spot runs interrupt-tolerant tasks "at up to a 70% discount off the regular Fargate price." Spot rates are set by AWS and adjust **gradually** with long-term supply/demand — no EC2-style auction spikes — so always pull the current rate from the Fargate pricing page.

- us-east-1 on-demand (Linux/x86): **$0.04048/vCPU-hr + $0.004445/GB-hr** → a 1 vCPU / 2 GB task = **$36.04/month** (730 hrs).
- Same task at the full 70% Spot discount ≈ **$10.81/month**.
- **10-task service:** all on-demand = $360.40/mo. With base 2 + weights 1:3 (4 on-demand + 6 Spot) = **$209.02/mo — a 42% cut**. Pushing weights to 1:9 (3 on-demand + 7 Spot) = $183.79/mo (−49%).
- Observability overhead: EventBridge rules on AWS service events are free; two CloudWatch alarms = $0.20/mo; SNS and a 30-day log group are pennies at this volume.

## Troubleshooting

1. **5xx spike during a reclaim.** Cause: deregistration delay still at the 300-second default, so the ALB drains longer than the 120-second SIGKILL deadline. Fix: set `deregistration_delay.timeout_seconds=90` and `stopTimeout=120`, redeploy, re-run the drain drill.
2. **`update-service` rejects the strategy ("capacity provider not associated with the cluster").** Cause: FARGATE/FARGATE_SPOT never attached to this cluster. Fix: `aws ecs put-cluster-capacity-providers` — and include every provider the cluster needs, because the call replaces the full list.
3. **App killed at stopTimeout without cleanup.** Cause: shell-form `ENTRYPOINT` makes `/bin/sh` PID 1, which swallows SIGTERM. Fix: exec-form JSON `ENTRYPOINT` (or an init process), then test locally with `docker kill -s TERM`.
4. **Tasks stuck in PROVISIONING; service event "unable to place a task."** Cause: Fargate Spot capacity temporarily unavailable — ECS retries automatically and never substitutes on-demand. Fix: if the floor is too low for your SLO, raise `base` or shift weight toward FARGATE; capacity typically returns without action.
5. **Task fails to launch on FARGATE_SPOT.** Cause: the task targets Windows — Fargate Spot supports Linux only (both x86_64 and ARM64/Graviton run on FARGATE_SPOT at platform version 1.4.0+, so a Graviton web tier can stack the ARM price-performance gain on top of the Spot discount). Fix: confirm the task is Linux and on PV 1.4.0 or LATEST; keep any Windows services on the FARGATE provider.

Production-ready in under 15 minutes: one Terraform apply, one forced deployment, zero downtime — then prove it with the drain drill.
