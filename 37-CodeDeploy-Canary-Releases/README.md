# CodeDeploy Canary Releases for Lambda and ECS: Alarm-Gated Traffic Shifting That Caps a Bad Deploy at 10% Blast Radius

Production-ready progressive delivery wiring — weighted Lambda aliases, ECS blue/green target-group swaps, and CloudWatch alarm gates that trigger automatic rollback in seconds—so a failed release costs you 300 requests instead of a 20,000-request outage.

## The Problem

Every all-at-once deploy is a bet with 100% of your traffic as the stake. Run the math for an API doing 1,000 requests/minute when a release ships a bug that 500s every call on the checkout path:

- **Before (all-at-once, human rollback):** the on-call page fires ~5 minutes in, the engineer confirms and finds the culprit in ~5 more, and the pipeline takes 10–15 minutes to redeploy the previous build. Total exposure ≈ 20 minutes × 1,000 req/min × 100% of traffic = **~20,000 failed requests, every user affected**.
- **After (`Canary10Percent5Minutes` + alarm gate):** CodeDeploy shifts 10% of traffic to the new version. An error-rate alarm set at 5% over 2 consecutive 1-minute datapoints trips ~2–3 minutes in, CodeDeploy stops the deployment and flips the alias weight back in seconds. Total exposure ≈ 3 minutes × 1,000 req/min × 10% = **~300 failed requests — a 98.5% reduction, and 90% of users never saw it**.

The mechanism is nearly free: CodeDeploy charges $0 for Lambda and ECS deployments, and the entire gate is six $0.10/month CloudWatch alarms. Most teams skip it anyway, because the wiring spans four services (CodeDeploy deployment groups, alias routing configs, target-group listeners, alarm dimensions) and one wrong dimension means the gate silently never fires.

This is a runtime delivery mechanism, not a CI promotion gate: production traffic itself is the test, the alarm is the judge, and rollback needs no human.

## Who This Is For

Startups shipping a Lambda API or an ECS Fargate service several times a week with no dedicated release engineer; teams that have been burned by a Friday deploy and now fear the deploy button; anyone graduating from "deploy and watch Slack" to measured progressive delivery.

## How to Use

1. Copy the System Prompt below into Kiro CLI, Claude Code, or Cursor (any assistant that can run AWS CLI commands works).
2. Replace the bracketed placeholders: `[FUNCTION_NAME]`, `[SERVICE_NAME]`, `[CLUSTER_NAME]`, `[ALB_ARN]`, `[REGION]`, `[PEAK_RPM]`, `[P99_MS]`.
3. Let the assistant run its verification pass first — it confirms the function, alias, ECS service, and deployment configs exist before generating a single line.
4. Review the generated SAM template, appspec, and hook code; deploy to staging with `sam deploy --config-env staging` (staging uses `AllAtOnce` so iteration stays fast).
5. Run the included `gameday.sh` against staging: it deploys a deliberately broken handler and prints the stop-on-alarm and rollback timeline as proof the gate works.

**Pick your deployment preference** (Lambda SAM type / ECS predefined config):

| Config | Traffic pattern | 100% at | Use when |
|---|---|---|---|
| `Canary10Percent5Minutes` / `CodeDeployDefault.ECSCanary10Percent5Minutes` | 10%, hold 5 min, then 100% | 5 min | Default for steady-traffic services |
| `Canary10Percent15Minutes` / `CodeDeployDefault.ECSCanary10Percent15Minutes` | 10%, hold 15 min | 15 min | Lower traffic — needs more datapoints |
| `Canary10Percent30Minutes` (Lambda only) | 10%, hold 30 min | 30 min | Spiky/low-RPS functions |
| `Linear10PercentEvery1Minute` / `CodeDeployDefault.ECSLinear10PercentEvery1Minutes` | +10% per minute | ~10 min | Gradual ramp, latency-sensitive |
| `Linear10PercentEvery10Minutes` (Lambda only) | +10% every 10 min | ~90 min | High-risk dependency or data-shape changes |
| `AllAtOnce` / `CodeDeployDefault.ECSAllAtOnce` | 100% immediately | 0 min | Dev/staging only — never production |

**Prerequisites**

- **Required Access:** IAM permissions for `codedeploy:*`, `lambda:GetAlias`/`UpdateAlias`/`PublishVersion`, `cloudwatch:PutMetricAlarm`, `ecs:DescribeServices`/`CreateService`, `elasticloadbalancing:*` on the target groups/listeners, and `iam:PassRole`. CodeDeploy service roles attach the managed policies `AWSCodeDeployRoleForLambda` (Lambda) and `AWSCodeDeployRoleForECS` (ECS).
- **Recommended Background:** you have deployed the Lambda function or ECS service at least once; basic familiarity with SAM or CDK; you know your service's normal error rate and p99 latency.
- **Tools Required:** AWS CLI v2, SAM CLI >= 1.100 (or CDK v2), and the AWS Documentation MCP server — the prompt instructs the assistant to use `aws_search_documentation` and `aws_read_documentation` to confirm current config names, plus `use_aws` (or shell AWS CLI) for the verification calls.

**Key Parameters:** deployment preference (`Canary10Percent5Minutes`), error-rate gate (5% over 2×1-min datapoints), p99 latency gate (`[P99_MS]`, default 2,000 ms), ECS bake window `terminationWaitTimeMinutes` (60, max 2,880), `TreatMissingData` (`notBreaching`), pre-traffic hook timeout (300 s), alarms per deployment group (10 max).

**Troubleshooting:** The first deployment never canaries — CodeDeploy needs an existing version (Lambda) or live target group (ECS) to shift traffic *from*, so SAM's initial deploy points the `live` alias straight at version 1. This is expected; the alarm gates apply from deploy #2 onward.

## System Prompt

```
# Progressive Delivery Architecture Request: Alarm-Gated Canary Releases with CodeDeploy

## Project Overview
Wire production-ready canary releases for two workloads in [REGION]:
1. Lambda API: function [FUNCTION_NAME], traffic served through alias `live`.
2. ECS Fargate service: [SERVICE_NAME] in cluster [CLUSTER_NAME], behind ALB [ALB_ARN].

Hard goal: no release ever exposes more than 10% of production traffic to an unverified version, and a failing release rolls back automatically in under 60 seconds with zero human action.

## Verify Reality First (before generating anything)
- Confirm resources exist: `aws lambda get-alias --function-name [FUNCTION_NAME] --name live` and `aws ecs describe-services --cluster [CLUSTER_NAME] --services [SERVICE_NAME]`.
- Check the ECS service's deploymentController. If it is ECS (rolling) rather than CODE_DEPLOY, STOP and report: the controller is immutable after creation and the service must be recreated.
- Confirm the ALB can host a second target group and a test listener on port 8443.
- Run `aws deploy list-deployment-configs` and confirm `CodeDeployDefault.ECSCanary10Percent5Minutes` is available in [REGION]; verify current config names against AWS documentation (aws_read_documentation) rather than memory.
- Check SAM CLI >= 1.100 (or CDK v2). If any check fails, report the failure and remediation steps; never generate against assumed state.

## Detailed Requirements

### 1. Lambda traffic shifting
- `AutoPublishAlias: live`; `DeploymentPreference` Type `Canary10Percent5Minutes`, parameterized so staging uses `AllAtOnce` and high-risk releases can switch to `Linear10PercentEvery10Minutes`.
- BeforeAllowTraffic hook: Python 3.13 function, 300 s timeout, fires 3 synthetic requests at the new version and reports `Failed` on any non-2xx via `put_lifecycle_event_hook_execution_status`. Grant the hook role `codedeploy:PutLifecycleEventHookExecutionStatus`.
- Auto-rollback on DEPLOYMENT_FAILURE, DEPLOYMENT_STOP_ON_ALARM, and DEPLOYMENT_STOP_ON_REQUEST.

### 2. ECS blue/green
- Two target groups; production listener :443, test listener :8443; deployment config `CodeDeployDefault.ECSCanary10Percent5Minutes`.
- Keep blue warm: `terminationWaitTimeMinutes: 60`, so rollback is a listener flip, not a task relaunch.
- AfterAllowTestTraffic hook smoke-tests green on :8443 before any production traffic shifts.

### 3. Alarm gates (the rollback wiring)
Attach every alarm to the deployment group alarmConfiguration (10 max):
- Lambda error rate: metric math Errors/Invocations with dimensions FunctionName=[FUNCTION_NAME], Resource=[FUNCTION_NAME]:live, threshold 5% for 2 of 2 one-minute datapoints, TreatMissingData=notBreaching.
- Lambda p99 Duration > [P99_MS] ms for 3 of 3 one-minute datapoints.
- ECS green target group: HTTPCode_Target_5XX_Count/RequestCount > 2% for 2 of 2; TargetResponseTime p99 > 2 s; UnHealthyHostCount >= 1 for 2 datapoints.

### 4. Cost ceiling
Added run-rate stays under $2/month: CodeDeploy traffic shifting is free for Lambda and ECS; six standard-resolution alarms cost $0.60/month; ECS doubles task count only during the shift plus the 60-minute bake.

### 5. Blast-radius math
Produce a before/after table at [PEAK_RPM] requests/minute: all-at-once bad deploy with a 20-minute human detect-and-rollback loop versus a 10% canary with a 2-minute alarm trip and sub-minute automatic rollback. Show failed-request counts and percentage reduction.

## Deliverables Requested
1. `template.yaml` (SAM preferred) covering both workloads, alarms, hooks, and IAM — CodeDeploy service roles using the AWSCodeDeployRoleForLambda and AWSCodeDeployRoleForECS managed policies.
2. `appspec.yaml` + `taskdef.json` for the ECS deployment group.
3. `hooks/pre_traffic.py` and `hooks/post_traffic.py`.
4. `gameday.sh` — deploys a deliberately broken handler, polls `aws deploy get-deployment`, and prints the stop-on-alarm and rollback event timeline.
5. `ROLLBACK-RUNBOOK.md` — the manual abort command (`aws deploy stop-deployment --auto-rollback-enabled`) and a post-rollback checklist.
6. `blast-radius.md` with the math from requirement 5.

## Acceptance Evidence
End with an Acceptance section proving the wiring works: the alias RoutingConfig showing weight 0.1 mid-deploy, deployment events showing DEPLOYMENT_STOP_ON_ALARM followed by rollback completion, and alarm state-transition timestamps. Map the design to the Well-Architected Operational Excellence and Reliability pillars. Output the files only, without any preamble.
```

## What You Get

1. **`template.yaml`** — SAM template with `AutoPublishAlias`, `DeploymentPreference`, hook wiring, all six alarms, and least-privilege IAM (CodeDeploy roles via `AWSCodeDeployRoleForLambda` / `AWSCodeDeployRoleForECS`).
2. **`appspec.yaml` + `taskdef.json`** — ECS blue/green deployment spec with lifecycle hook mappings.
3. **`hooks/pre_traffic.py` / `hooks/post_traffic.py`** — synthetic smoke tests that report back through `put_lifecycle_event_hook_execution_status`.
4. **Six CloudWatch alarm definitions** — Lambda error rate + p99, ECS 5xx ratio + p99 + unhealthy hosts, all dimensioned to the canary side.
5. **`gameday.sh`** — one command that ships a broken build and captures the automatic rollback as evidence.
6. **`ROLLBACK-RUNBOOK.md`** — the 2 AM page answer: how to abort mid-shift, what state to expect after rollback, what to check before retrying.
7. **`blast-radius.md`** — your before/after failed-request math at your real `[PEAK_RPM]`.

## Example Output

```yaml
CheckoutFunction:
  Type: AWS::Serverless::Function
  Properties:
    AutoPublishAlias: live
    DeploymentPreference:
      Type: Canary10Percent5Minutes
      Alarms:
        - !Ref ErrorRateGt5Pct
        - !Ref P99DurationGt2s
      Hooks:
        PreTraffic: !Ref PreTrafficHook
```

Game-day evidence from `gameday.sh`:

```
14:29:55  d-4K7XJ2A9B  CREATED      config=CodeDeployDefault.LambdaCanary10Percent5Minutes
14:30:12  shift start  live -> v41: 90% / v42: 10%
14:32:08  ALARM        checkout-ErrorRateGt5Pct (12.4% > 5.0%)
14:32:09  STOPPED      DEPLOYMENT_STOP_ON_ALARM, rollback initiated
14:32:41  ROLLED BACK  live -> v41: 100%  |  blast radius: 312 requests (10% of traffic for 2m56s)
```

## AWS Services Used

AWS CodeDeploy, AWS Lambda, Amazon ECS (Fargate), Elastic Load Balancing (Application Load Balancer), Amazon CloudWatch, AWS SAM / AWS CloudFormation, AWS IAM, Amazon SNS (alarm notifications).

## Well-Architected Alignment

- **Operational Excellence:** implements the safe-deployment best practice directly — small reversible changes, automated rollback, a game-day script, and a written rollback runbook.
- **Reliability:** caps change-failure blast radius at 10% of traffic; blue stays warm for 60 minutes so ECS rollback is a listener flip; rollback requires no human in the loop.
- **Cost Optimization:** the whole mechanism adds under $2/month; CodeDeploy itself is free for Lambda and ECS targets; no permanently duplicated fleet.
- **Security:** CodeDeploy service roles use the scoped AWS managed policies; hook functions get only `codedeploy:PutLifecycleEventHookExecutionStatus` plus invoke rights on the target.
- **Performance Efficiency:** the p99 latency gate catches performance regressions while only 10% of users are exposed.

## Cost Notes

- **CodeDeploy:** $0 — there is no charge for deployments to Lambda or ECS.
- **CloudWatch alarms:** 6 standard-resolution metric alarms × $0.10 = **$0.60/month**.
- **Lambda:** weighted aliases split traffic, they don't duplicate it — $0 extra.
- **ECS:** blue and green run in parallel during shift + bake. Four 0.25 vCPU / 0.5 GB Fargate tasks ≈ $0.0123/task-hour (us-east-1), so a deploy with a 60-minute bake adds ≈ **$0.06**; 20 deploys/month ≈ $1.23.
- **ALB:** you already pay ~$16.43/month for the load balancer; the second target group and test listener add $0.
- **Total added run-rate: under $2/month** for a 98.5% cut in bad-deploy blast radius.

## Troubleshooting

1. **Deployment stops on alarm seconds after starting, service is healthy.** Cause: a gate alarm was already in ALARM before the deploy began (stale baseline), or an ECS alarm is dimensioned to the blue target group instead of green. Fix: run `aws cloudwatch describe-alarms --state-value ALARM` pre-deploy; re-dimension ECS alarms to the green target group.
2. **A bad version got promoted even though it was erroring.** Cause: low canary traffic produced empty 1-minute metric bins and `TreatMissingData: notBreaching` never tripped. Fix: lengthen the hold to `Canary10Percent30Minutes` (Lambda) / `ECSCanary10Percent15Minutes` (ECS) and lean on the BeforeAllowTraffic synthetic hook for low-RPS services.
3. **ECS service won't accept the CODE_DEPLOY deployment controller.** Cause: `deploymentController` is immutable after service creation. Fix: create a replacement service with `CODE_DEPLOY` set, shift traffic via the ALB, then delete the old service.
4. **Deployment hangs at the hook step for an hour, then fails.** Cause: the hook function never reported back — it must call `PutLifecycleEventHookExecutionStatus` with the `DeploymentId` and `LifecycleEventHookExecutionId` from its event payload, and its role needs that permission. Fix: add the API call and the IAM allow; set the hook timeout to 300 s so failures surface fast.
5. **First deploy went straight to 100%.** Cause: expected behavior — there is no prior version or target group to shift from. Fix: nothing; verify the gates with `gameday.sh` on deploy #2.

A failed release should be a three-minute, 10% blip you read about in a Slack bot message — not an incident bridge. This kit is deployable to staging in under 15 minutes, and the game-day script proves the rollback before production ever depends on it.
