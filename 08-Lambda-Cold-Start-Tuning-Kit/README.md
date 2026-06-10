# Lambda Cold-Start & Cost Tuning Kit: Power Tuning, SnapStart & Provisioned-Concurrency Break-Even (Measured, Not Guessed)

Tune AWS Lambda memory, cold starts, and concurrency on measured evidence—the Power Tuning state machine, a real SnapStart eligibility table, break-even math in dollars—so you cut p99 latency without overpaying for "just add memory."

## The Problem

"Just bump the memory to 1769 MB" is the most expensive guess in serverless. Memory is the only Lambda knob, and it sets CPU too—1,769 MB buys exactly one vCPU—so the curve is non-linear: a 128 MB function is often *slower AND more expensive per invocation* than a 512 MB one because it runs 4x longer. Teams pick numbers by vibes, then overpay 3-4x or eat 2-8 second cold starts on a Java/Spring API.

Concrete numbers: a 512 MB Java function with a 6-second init hits ~4% cold starts at a p99 of 5.8s. Engineers reach for provisioned concurrency—$0.0000041667/GB-s (us-east-1), $10.95/month for ONE always-warm 1 GB instance, before any invocation. Many provision 10 and quietly add $109.50/month for latency SnapStart kills for free on Java. The fix is never "more memory." It is *measure, then choose*.

## Who This Is For

- Startup engineers running latency-sensitive Lambda APIs (API Gateway, Function URLs, ALB) with p99 cold-start spikes.
- Platform and FinOps owners who must defend a memory and concurrency decision with an artifact, not an opinion.
- Java/Spring Boot, Python, and .NET teams choosing between SnapStart, provisioned concurrency, and right-sizing.

## How to Use

1. Open Claude Code, Kiro CLI, or Amazon Q Developer CLI in the repo holding your Lambda (SAM, CDK, Terraform, or Serverless Framework).
2. Connect the AWS Documentation MCP server so the assistant verifies SnapStart support and pricing instead of guessing.
3. Paste the System Prompt below. Replace the bracketed placeholders: [FUNCTION_ARN], [RUNTIME], [REGION], [MONTHLY_INVOCATIONS], [CURRENT_MEMORY_MB], [P99_TARGET_MS].
4. Let it deploy the `aws-lambda-power-tuning` SAR app, run the Step Functions execution, and return the visualization URL plus recommended memory.
5. Review the eligibility table and break-even math, then apply with the emitted AWS CLI commands.

Prerequisites:
- Required Access: An IAM principal with `lambda:GetFunction`, `lambda:UpdateFunctionConfiguration`, `lambda:PublishVersion`, `lambda:PutProvisionedConcurrencyConfig`, `states:StartExecution`/`states:DescribeExecution`, `serverlessrepo:CreateCloudFormationChangeSet`, and `cloudformation:*` scoped to the power-tuning stack. `AWSLambda_FullAccess` covers the Lambda actions but NOT Step Functions, CloudFormation, or SAR—add those explicitly; tighten before production.
- Recommended Background: Lambda init phase vs. handler duration, Step Functions, and reading `Init Duration` / `Duration` from REPORT lines.
- Tools Required: AWS Documentation MCP server (`aws___read_documentation`, `aws___search_documentation`), AWS CLI v2, the Power Tuning SAR app v4.x.

Key Parameters: powerValues (128/256/512/1024/1536/1769/3008 MB—1769 MB = 1 full vCPU), num=50 invocations per power, strategy=balanced with balancedWeight=0.5, payload (representative event JSON), parallelInvocation=true, autoOptimize=false (review before apply), PC standing charge $0.0000041667/GB-s, SnapStart ApplyOn=PublishedVersions.

Troubleshooting: If Power Tuning fails with `Lambda.TooManyRequestsException`, reserved concurrency sits below `num`—expected; parallel sampling exceeds it. Raise reserved concurrency to 50 for the test or set parallelInvocation=false.

## System Prompt

```
# AWS Lambda Cold-Start & Cost Tuning — Measured Optimization Request

Act as a senior AWS serverless performance engineer. Produce a production-ready,
evidence-backed Lambda tuning decision. NEVER recommend a memory size, SnapStart,
or provisioned concurrency from intuition: every recommendation MUST cite a
measured Power Tuning result or a dollar calculation shown in full.

## Target Function
- Function ARN: [FUNCTION_ARN]
- Runtime: [RUNTIME] (e.g., java21, python3.13, dotnet8, nodejs22.x)
- Region: [REGION]
- Monthly invocations: [MONTHLY_INVOCATIONS]
- Current memory (MB): [CURRENT_MEMORY_MB]
- p99 latency SLO (ms): [P99_TARGET_MS]

## Step 0 — Verify Reality FIRST
1. Call lambda get-function; read the LIVE Runtime, MemorySize, Timeout,
   EphemeralStorage, FileSystemConfigs, Architectures, PackageType. If a
   placeholder contradicts the API, trust the API and say so.
2. Via the AWS Documentation MCP (aws___read_documentation), CONFIRM today's
   SnapStart support for this runtime (Java 11+, Python 3.12+, .NET 8+ —
   verify, never assume) and the Region prices used in Step 3.
3. SnapStart blockers on the live config: container images, provisioned
   concurrency on the same version, Amazon EFS, or ephemeral storage > 512 MB.
4. If any fact cannot be verified, STOP and report which one. Removal beats
   invention.

## Step 1 — Deploy & Run Power Tuning (the acceptance artifact)
1. Deploy the aws-lambda-power-tuning SAR app via a CloudFormation change set;
   output the StateMachineArn.
2. Start a Step Functions execution with exactly:
   { "lambdaARN": "[FUNCTION_ARN]",
     "powerValues": [128, 256, 512, 1024, 1536, 1769, 3008],
     "num": 50, "payload": <REPRESENTATIVE_EVENT_JSON>,
     "parallelInvocation": true, "strategy": "balanced",
     "balancedWeight": 0.5, "autoOptimize": false }
3. Poll states describe-execution until SUCCEEDED; emit the FULL output JSON
   verbatim: power, cost, duration, stateMachine.executionCost, lambdaCost,
   visualization. The lambda-power-tuning.show URL is the acceptance artifact —
   never omit it. Recommend the returned power; never override it.

## Step 2 — SnapStart Eligibility Table
Emit a table: Option | Eligible for THIS runtime? | Cold-start effect | Extra
cost | Constraint. Rows: SnapStart, Provisioned Concurrency, Right-size only.
Mark SnapStart INELIGIBLE for Node.js, Ruby, OS-only, and container-image
functions, and wherever a Step 0 blocker fired. SnapStart activates ONLY on
published versions and aliases — never $LATEST — via
--snap-start ApplyOn=PublishedVersions then publish-version.

## Step 3 — Provisioned-Concurrency Break-Even (show every number)
- PC standing cost = memory_GB x $0.0000041667/GB-s x 3,600 x 730 h x count
  (1 GB x 1 instance ≈ $10.95/month before any invocation). PC duration bills
  at $0.0000097222/GB-s vs $0.0000166667/GB-s on-demand (us-east-1; confirm
  for [REGION]).
- Compare SnapStart: $0 extra on Java; on Python 3.12+/.NET 8+, cache
  $0.0000015046/GB-s (3-hour minimum/version) plus restore $0.0001397998/GB
  restored.
- Close with one rule: prefer SnapStart when eligible; buy provisioned
  concurrency only when measured p99 cold start violates [P99_TARGET_MS] AND
  traffic is steady enough that the standing charge beats the cold-start tax.
  Show the monthly crossover.

## Error Management
- Lambda.TooManyRequestsException → reserved concurrency below num=50. Raise
  it for the test or set parallelInvocation=false; never tune around a
  throttle silently.
- Execution RUNNING far past expectations → timeout x num x 7 powers
  compounds. Cut num to 20 or test fewer powers.
- SAR deploy fails → missing serverlessrepo:CreateCloudFormationChangeSet or
  cloudformation actions. Print the exact missing action; never widen beyond it.
- SnapStart enable rejected → re-run the Step 0 blocker checklist; report
  which constraint fired.

## Output / Acceptance Evidence
Produce, in order: (1) the verbatim Power Tuning output JSON + visualization
URL; (2) the eligibility table; (3) the break-even math, every dollar shown;
(4) exact apply commands — aws lambda update-function-configuration
--memory-size N; if chosen, --snap-start ApplyOn=PublishedVersions then
publish-version, or put-provisioned-concurrency-config; (5) a Verification
section: CloudWatch Duration p99 plus Init Duration from REPORT log lines via
Logs Insights, with the expected post-change delta in ms and $/month. Use
exact resource names, flags, and prices; flag any unverifiable
number rather than inventing it. The final answer must be production-ready and
reviewable by a staff engineer in under 5 minutes.
```

## What You Get

1. A deployed Power Tuning state machine (CloudFormation stack from the `aws-lambda-power-tuning` SAR app) with its StateMachineArn.
2. The verbatim output JSON—`power`, `cost`, `duration`, `stateMachine.executionCost`, `lambdaCost`, and the `visualization` URL—the acceptance artifact.
3. A SnapStart eligibility table for your exact runtime, with the disqualifying constraint for each option.
4. Break-even math in real dollars: the $10.95/month-per-GB standing charge, the SnapStart comparison, and the monthly crossover.
5. Copy-paste AWS CLI commands to apply the memory, enable SnapStart (`ApplyOn=PublishedVersions` + `publish-version`), or set provisioned concurrency.
6. A Verification section: `Duration` p99 in CloudWatch, `Init Duration` from REPORT lines via Logs Insights, and the expected delta.

## Example Output

Power Tuning result (us-east-1, java21): recommended power = 1024 MB. At 512 MB, avg duration 980 ms, cost $8.18/1M; at 1024 MB, 410 ms, $6.95/1M — faster AND cheaper. Visualization: https://lambda-power-tuning.show/#<base64-state>. SnapStart: ELIGIBLE (Java 21); p99 cold start drops from 5.8s to ~0.9s at $0 extra. Provisioned concurrency NOT recommended: 10 x 1 GB = $109.50/month standing for a cold start SnapStart fixes free.

## AWS Services Used

AWS Lambda, AWS Lambda Power Tuning (Serverless Application Repository), AWS Step Functions, AWS CloudFormation, Amazon CloudWatch, AWS IAM, AWS CLI.

## Well-Architected Alignment

- Cost Optimization: memory chosen on the measured cost/latency curve; break-even quantified before spend; free SnapStart preferred on Java.
- Performance Efficiency: tunes the one knob that matters (memory = CPU; 1769 MB = 1 vCPU) with data across seven power values.
- Operational Excellence: a reviewable artifact (state machine output + visualization URL) plus a CloudWatch verification checklist.
- Reliability: `autoOptimize=false` keeps a human review before production changes; throttles surface instead of being tuned around.
- Security: the tuning principal is scoped to named IAM actions, not broad admin; changes apply via reviewed CLI commands.

## Cost Notes

- The Power Tuning run: 7 powers x 50 invocations = 350 test calls—a few cents, within the Lambda free tier (1M requests, 400,000 GB-s/month; $0.20/1M after).
- Provisioned concurrency: $0.0000041667/GB-s standing = $10.95/month per always-warm 1 GB instance (us-east-1); duration $0.0000097222/GB-s vs on-demand $0.0000166667/GB-s.
- SnapStart: $0 extra on Java managed runtimes. On Python 3.12+/.NET 8+: cache $0.0000015046/GB-s (3-hour minimum per published version) + restore $0.0001397998/GB.
- Delete unused published versions to stop SnapStart cache charges; over-provisioned memory on a hot function is often the largest avoidable Lambda line item.

## Troubleshooting

- `Lambda.TooManyRequestsException` during Power Tuning. Cause: reserved concurrency below `num` (50) blocks parallel sampling. Fix: raise it to >=50 for the test or set `parallelInvocation=false`.
- SnapStart enable has no effect. Cause: it applies only to published versions/aliases, never $LATEST. Fix: set `--snap-start ApplyOn=PublishedVersions`, then `lambda publish-version`.
- Recommended memory raises the bill. Cause: `strategy` defaulted to `speed`. Fix: re-run with `strategy=balanced` (or `cost`).
- p99 cold start still spikes after SnapStart. Cause: heavy post-restore handler work or stale connections after resume. Fix: move init outside the handler; re-establish SDK/DB connections on restore.
- Provisioned concurrency costs more than expected. Cause: instances bill 24/7, invoked or not. Fix: compute the break-even, schedule with Application Auto Scaling, or switch to SnapStart.
