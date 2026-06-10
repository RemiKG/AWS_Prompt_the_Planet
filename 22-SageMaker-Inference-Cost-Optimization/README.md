# SageMaker Inference Cost Optimizer: Serverless vs Real-Time vs MME Decision Kit

Pick the right SageMaker hosting option with real price math and scale-to-zero — so your inference bill tracks actual traffic instead of bleeding $1,000+/month per idle ml.g5 endpoint.

## The Problem

A startup deployed three fine-tuned models on three always-on `ml.g5.xlarge` real-time endpoints. At the us-east-1 rate of $1.408/hour each, that is ~$3,084/month (730 hours) for endpoints serving traffic four hours a day. The team heard "just use Bedrock," abandoned SageMaker, and stranded models it had already paid to train. The real fix is matching the hosting option to the traffic shape. SageMaker offers four options with wildly different cost curves: Serverless Inference (scales to zero, billed by the millisecond, no GPU), Real-Time (lowest latency; scale-to-zero via Inference Components since re:Invent 2024), Multi-Model Endpoints (many models on one LRU-cached instance), and Asynchronous (1 GB payloads, scales to zero). Choosing wrong costs 5-20x. This prompt produces the decision table, price math, and deployable config so you choose with numbers, not vibes.

## Who This Is For

Founders and ML engineers running 1-50 custom models (classification, embeddings, fine-tuned LLMs, CV) on SageMaker who see a flat hosting bill regardless of traffic, want idle spend gone without rewriting to Bedrock, and need a decision they can defend to a cost-conscious CTO.

## How to Use

1. Open Kiro CLI, Claude Code, or Amazon Q Developer in your project directory.
2. Connect the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) to verify limits and pricing against live docs.
3. Paste the System Prompt below into a new conversation.
4. Replace every `[BRACKETED]` placeholder: model count and sizes, requests/day, peak concurrency, p99 SLA, payload size, GPU requirement, region, and budget.
5. Let it verify reality first (existing endpoints, instance availability, quotas), then generate the decision table and IaC.
6. Review the cost table and verification section before you `terraform apply`.

Prerequisites:
- Required Access: An IAM principal with `sagemaker:CreateModel`, `sagemaker:CreateEndpointConfig`, `sagemaker:CreateEndpoint`, `sagemaker:CreateInferenceComponent`, `application-autoscaling:RegisterScalableTarget`, and `cloudwatch:PutMetricAlarm`, plus a SageMaker execution role with `s3:GetObject` on the model-artifact bucket.
- Recommended Background: One prior SageMaker endpoint deployment and basic Terraform or boto3.
- Tools Required: AWS CLI v2, Terraform >= 1.6 (or AWS CDK v2), the AWS Documentation MCP server, and credentials for the target account.

Key Parameters: serverless memory tiers 1024/2048/3072/4096/5120/6144 MB, MaxConcurrency 1-200, ProvisionedConcurrency <= MaxConcurrency, 4 MB payload / 60 s timeout (serverless), ManagedInstanceScaling MinInstanceCount 0 (real-time scale-to-zero), cooldowns 60 s out / 300 s in, MME LRU eviction bounded by instance memory.

Troubleshooting: If the assistant proposes a GPU serverless endpoint, that is a hard stop — Serverless Inference supports no GPUs, MME, VPC config, Model Monitor, or data capture; route those workloads to Real-Time or MME.

## System Prompt

```
# SageMaker Inference Hosting — Cost-Optimized Architecture Design Request

Act as a senior AWS ML infrastructure engineer. Recommend and generate the
production-ready SageMaker hosting configuration that minimizes cost for my
traffic shape. Do NOT default to always-on real-time endpoints.

## Verify Reality First (before generating anything)
1. Via the AWS Documentation MCP server, confirm CURRENT Serverless Inference
   limits: memory tiers 1024-6144 MB, MaxConcurrency up to 200 per endpoint,
   the per-Region concurrency quota, the 4 MB payload / 60 s timeout, and the
   exclusions (no GPU, MME, VPC config, Model Monitor, or data capture). Cite
   the doc page for every number.
2. Confirm real-time scale-to-zero needs Inference Components with
   ManagedInstanceScaling MinInstanceCount=0 (GA since re:Invent 2024);
   target tracking on SageMakerInferenceComponentInvocationsPerCopy
   (min_capacity=0) scales in after ~15 min idle; scale-out FROM zero needs
   a step-scaling policy on NoCapacityInvocationFailures.
3. Run `aws sagemaker list-endpoints` / `describe-endpoint` on my existing
   endpoints — right-size from reality, not assumptions.
4. Confirm my candidate instance types exist in [REGION]; flag any that don't.
5. Check Service Quotas for the instance type and serverless concurrency.
If any check fails, STOP and say what to fix. Never fabricate a price, quota,
or instance type.

## My Workload
- Models: [NUMBER] models, [X] GB each, framework [PYTORCH/TF/XGBOOST].
- Traffic: [N] requests/day, peak [C] concurrent, pattern [SPIKY/STEADY/BURSTY].
- Latency SLA: p99 < [L] ms. GPU required: [YES/NO]. Payload: [P] MB.
- Region: [REGION]. Monthly inference budget: $[BUDGET].

## Required Output

### 1. Decision Table
Compare Serverless, Real-Time with scale-to-zero, Multi-Model Endpoints
(MME), and Async Inference across: cold-start behavior, scale-to-zero
mechanism, GPU support, payload limit, max models, best traffic shape, and
ESTIMATED MONTHLY COST for MY numbers with price math shown per row.
Recommend ONE option; state the dollar delta vs always-on real-time.

### 2. Price Math
- Always-on baseline: instance hourly rate x 730 hours x instance count.
- Serverless: (memory-tier $/second x avg duration x monthly requests) +
  (per-GB processing fee x payload GB). Idle cost is $0.
- MME: ONE instance x 730 hours for ALL [NUMBER] models vs N endpoints;
  call out the LRU cache-eviction reload risk.
- Provisioned Concurrency: break-even volume where it beats on-demand.
Round to whole dollars and state every assumption.

### 3. Cold-Start Mitigation
- Serverless: memory tier >= model size, MaxConcurrency = [C]; enable
  Provisioned Concurrency (billed even at idle) only if the p99 SLA demands
  it; monitor ModelSetupTime and OverheadLatency.
- Real-time scale-to-zero: the first request after scale-in fails until a
  copy starts — pair the NoCapacityInvocationFailures alarm with an
  aws_appautoscaling_scheduled_action before known traffic windows.
- MME: watch ModelCacheHit / ModelLoadingWaitTime; size memory to the hot set.

### 4. Infrastructure as Code
Generate Terraform (>= 1.6) or boto3 for the recommended option:
- Serverless: aws_sagemaker_endpoint_configuration with serverless_config
  { memory_size_in_mb, max_concurrency, provisioned_concurrency }.
- Real-time: ManagedInstanceScaling MinInstanceCount=0, boto3
  create_inference_component, aws_appautoscaling_target (min_capacity=0,
  dimension sagemaker:inference-component:DesiredCopyCount), target tracking
  plus the scale-from-zero alarm.
- MME: model with Mode=MultiModel on the S3 artifact prefix — all models
  MUST share one container.
Tag every resource Environment, Owner, CostCenter.

### 5. Guardrails
- AWS Budgets monthly budget at $[BUDGET] with 80% and 100% alerts.
- CloudWatch alarms on Invocation4XXErrors and p99 ModelLatency.
- Cooldowns: scale-out 60 s, scale-in 300 s.

### 6. Verification & Acceptance — emit evidence it works
- `aws sagemaker describe-endpoint` returns EndpointStatus=InService.
- A sample `aws sagemaker-runtime invoke-endpoint` call + expected output.
- CloudWatch evidence of scale-to-zero (copy count 0 at idle) and the
  expected cold-start range.
- One-line teardown (`aws sagemaker delete-endpoint`) to stop billing fast.

## Error Management
- ResourceLimitExceeded — Cause: endpoint or concurrency quota. Resolution:
  name the exact quota; emit the Service Quotas increase request.
- Serverless rejected for GPU/MME/VPC — Cause: unsupported feature.
  Resolution: hard stop; reroute to Real-Time or MME.
- Endpoint stuck in Creating — Cause: container or artifact failure.
  Resolution: read /aws/sagemaker/Endpoints/<name> logs.
- Copies never reach 0 — Cause: stray traffic or health pings. Resolution:
  inspect the scale-in alarm; silence the source.

## Rules
- Match hosting to traffic shape; never always-on GPU for spiky low-volume
  traffic. If GPU=YES, exclude Serverless.
- Every number must trace to a doc, a quota check, or a stated assumption.
- Output the decision table first, then IaC, then verification. No preamble.
The result must be production-ready, cheaper than my always-on baseline by a
stated dollar amount, and deployable in under 15 minutes.
```

## What You Get

- A four-option decision table scored on cold start, scale-to-zero mechanism, GPU support, payload limit, max models, traffic fit, and estimated monthly cost for YOUR numbers.
- Price math with every assumption stated, the dollar delta vs always-on, and the Provisioned Concurrency break-even volume.
- Cold-start mitigation: memory-tier sizing, Provisioned Concurrency, scheduled scale-out, ModelSetupTime/OverheadLatency monitoring.
- Terraform (or boto3): serverless endpoint config, or Inference Component + auto-scaling target with `min_capacity = 0` plus the NoCapacityInvocationFailures scale-from-zero alarm, or MME model with `Mode=MultiModel`.
- AWS Budgets budget with 80%/100% alerts plus CloudWatch alarms on 4XX errors and p99 ModelLatency.
- Verification: `describe-endpoint` InService check, sample `invoke-endpoint` call, scale-to-zero evidence, one-line teardown.

## Example Output

Recommendation: Multi-Model Endpoint on one ml.m5.xlarge. Decision table excerpt —
Serverless: scale-to-zero YES, GPU NO, ~$41/mo for 50k req/day at 2048 MB; Real-Time
3x ml.g5.xlarge always-on: ~$3,084/mo; MME 1x ml.m5.xlarge: $0.23/hr x 730 =
~$168/mo for all 3 models (LRU eviction risk noted). Delta vs always-on: about
-$2,916/month (-94%). Verification: describe-endpoint returns EndpointStatus
InService; idle copy count drops to 0 after ~15 minutes of zero traffic.

## AWS Services Used

Amazon SageMaker (Serverless Inference, Real-Time Inference, Multi-Model Endpoints, Asynchronous Inference, Inference Components), Application Auto Scaling, Amazon CloudWatch, AWS Budgets, AWS IAM, Amazon S3, AWS Cost Explorer.

## Well-Architected Alignment

- Cost Optimization: hosting matched to traffic shape, scale-to-zero at idle, models consolidated onto one MME instance, AWS Budgets ceiling with 80%/100% alerts.
- Operational Excellence: alarms on errors and latency, tagged resources, one-command teardown.
- Performance Efficiency: memory-tier and Provisioned-Concurrency sizing against a stated p99 SLA.
- Reliability: target-tracking auto-scaling with explicit cooldowns, a scale-from-zero alarm, a verified InService gate.
- Security: least-privilege execution role and scoped S3 access to the model-artifact bucket.

## Cost Notes

Real us-east-1 rates used in the math: ml.m5.large ~$0.115/hr, ml.m5.xlarge ~$0.23/hr, ml.g5.xlarge ~$1.408/hr (730 hrs/month). Three always-on ml.g5.xlarge = ~$3,084/month; one ml.m5.xlarge MME hosting all three models = ~$168/month, a ~94% cut. Serverless bills compute by the millisecond per memory tier plus a per-GB data fee — $0 at idle. Provisioned Concurrency bills every second it is configured, even idle, so it only pays off above the break-even volume the prompt computes. Verify current prices on the SageMaker pricing page for your region.

## Troubleshooting

- GPU on Serverless fails. Cause: no GPU, MME, VPC config, Model Monitor, or data capture on Serverless Inference. Fix: route those workloads to Real-Time or MME.
- Tight p99 SLA missed at low traffic. Cause: serverless cold start, visible in ModelSetupTime and OverheadLatency. Fix: Provisioned Concurrency sized to peak, or a scheduled scale-out before traffic windows.
- MME latency spikes. Cause: LRU eviction reloading a cold model from S3 (watch ModelCacheHit, ModelLoadingWaitTime). Fix: size instance memory to the working set, or split hot models out.
- ResourceLimitExceeded on create. Cause: an endpoint quota or the per-Region serverless concurrency quota. Fix: request an increase via Service Quotas; delete unused endpoints.
- Endpoint never scales to zero, or the first request after scale-in fails. Cause: classic auto-scaling keeps minimum instances at 1; at zero copies there is no traffic metric. Fix: Inference Components with ManagedInstanceScaling MinInstanceCount=0, scalable target min_capacity = 0, and the NoCapacityInvocationFailures step-scaling alarm to scale 0 to 1.
