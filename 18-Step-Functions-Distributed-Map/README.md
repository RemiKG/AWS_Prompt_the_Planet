# Process a Million Files Serverless: Step Functions Distributed Map with Failure Tolerance

Fan out 1,000,000 S3 objects across 10,000 parallel Lambda children with batching and a per-run failure budget—so one poison-pill record never fails the job and orchestration costs cents, not server hours.

## The Problem

You have 1,000,000 objects in S3 to transform, and you reach for a Step Functions inline `Map`. It dies—inline `Map` keeps every iteration inside one execution:

| Limit | Inline Map | Distributed Map |
|---|---|---|
| State payload | 256 KiB shared by the run | 256 KiB per child |
| Event history | 25,000 events total | 25,000 per child |
| Concurrency | ~40 iterations | 10,000 parallel children |
| Input source | execution input only | S3 direct: `listObjectsV2`, CSV/JSON, Inventory |

At a million items you blow through all three in seconds—`States.DataLimitExceeded`. The hand-rolled alternative (dispatcher, queue, DynamoDB failure tracking, replay script) is days of glue code, and when item 814,236 throws, the run stalls with no aggregated manifest.

**Distributed Map** reads S3 directly, gives each batch its own child execution, runs 10,000 in parallel, and writes a per-status manifest back to S3. The trap: tutorials omit `ItemBatcher`, `ToleratedFailurePercentage`, and a DLQ—you melt a downstream API or fail the whole run on one bad record. This prompt generates the production-ready definition with all three guardrails plus the redrive recovery path.

## Who This Is For

- Data and platform engineers running batch transforms (thumbnailing, PII redaction, re-indexing) over S3 at 100K-10M+ object scale.
- Startups that hit the inline `Map` 256 KiB / 40-concurrency wall and need to graduate without EMR or a batch cluster.
- Anyone who needs a failure budget so one corrupt object never nukes an 18-hour job.

## How to Use

1. Open Kiro CLI, Claude Code, or Amazon Q in the repo with your processing code.
2. Enable the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) so the assistant verifies the ASL schema first.
3. Paste the entire System Prompt below into a new session.
4. Replace every bracketed placeholder: `[SOURCE_BUCKET]`, `[SOURCE_PREFIX]`, `[RESULTS_BUCKET]`, `[PROCESSOR_FUNCTION_ARN]`, `[REGION]`, `[ITEM_COUNT]`.
5. Let it run the verification checks, then emit the state machine, IAM, DLQ, and deploy script.
6. Deploy and start one Map Run against a 1,000-object test prefix before the full 1,000,000.

Prerequisites
- Required Access: an IAM principal allowed `states:CreateStateMachine`, `iam:CreateRole`/`PutRolePolicy`, S3 read/write on both buckets, `sqs:CreateQueue`, and `lambda:InvokeFunction` on the processor.
- Recommended Background: comfort reading Amazon States Language (ASL) JSON; know your downstream throughput ceiling.
- Tools Required: AWS CLI v2, Kiro CLI / Claude Code / Amazon Q, the AWS Documentation MCP server.

Key Parameters: MaxConcurrency (1000, never above account Lambda concurrency), child ExecutionType (EXPRESS under 5 min/item, STANDARD above), MaxItemsPerBatch (100) / MaxInputBytesPerBatch (262144 ceiling), ToleratedFailurePercentage (1) / ToleratedFailureCount (1000)—AWS defaults both to 0, Lambda Retry (3 attempts, 2s, 2.0 backoff), 512 MB processor memory, log level ERROR.

Troubleshooting: If the parent state machine is created as Express type, deployment fails even with Express children—Distributed Map runs only inside a **Standard** parent.

## System Prompt

```
# Step Functions Distributed Map — Production Build Request

## Project Overview
Generate a production-ready AWS Step Functions Distributed Map state machine that processes [ITEM_COUNT] objects under s3://[SOURCE_BUCKET]/[SOURCE_PREFIX] by invoking Lambda [PROCESSOR_FUNCTION_ARN] in [REGION], writing a per-status manifest to s3://[RESULTS_BUCKET]. Output Amazon States Language (ASL) JSON, least-privilege IAM, and deploy commands. No preamble.

## Verify Reality FIRST
You WILL NOT emit a state machine until you have:
1. Confirmed the current ASL Distributed Map schema via the AWS Documentation MCP (aws_read_documentation). Never invent fields.
2. Confirmed both buckets exist in [REGION] (aws s3api head-bucket) and read the processor's timeout and memory (aws lambda get-function-configuration).
3. Read the account Lambda concurrency (aws lambda get-account-settings) and set MaxConcurrency at or below it (default 1,000); the 10,000 ceiling against a 1,000 limit throttles every invoke.
If any check fails, STOP, name the missing resource, and print the command that creates it. Hard stop.

## Detailed Requirements

### 1. Map mode and limits
- Parent MUST be type STANDARD; Distributed Map is not supported in an Express parent. Hard stop.
- "ProcessorConfig": { "Mode": "DISTRIBUTED", "ExecutionType": "EXPRESS" } when items finish under 5 minutes; STANDARD children for longer items or per-item audit history.
- Add a "Label" for identifiable Map Runs.

### 2. ItemReader and ResultWriter
- ItemReader: "arn:aws:states:::s3:listObjectsV2" for a prefix; s3:getObject with ReaderConfig InputType CSV or JSON for one large file; MANIFEST for S3 Inventory.
- ResultWriter: "arn:aws:states:::s3:putObject" to s3://[RESULTS_BUCKET]—Step Functions writes manifest.json plus per-status SUCCEEDED/FAILED/PENDING files. The manifest is mandatory.

### 3. Batching
- ItemBatcher MaxItemsPerBatch: 100 (1,000,000 items -> 10,000 children). Cap MaxInputBytesPerBatch at or below the 262,144-byte hard ceiling. Pass run constants via BatchInput.

### 4. Failure tolerance and error management (the core ask)
- Set ToleratedFailurePercentage: 1 AND ToleratedFailureCount: 1000. AWS defaults both to 0—one bad item fails the entire Map Run. With both set, States.ExceedToleratedFailureThreshold fires only when either budget is breached.
- Lambda Task Retry on Lambda.TooManyRequestsException, Lambda.ServiceException, States.TaskFailed: IntervalSeconds 2, MaxAttempts 3, BackoffRate 2.0.
- Catch exhausted failures into a state that sends failed object keys to the SQS DLQ—no failed item vanishes silently.
- Document recovery: aws stepfunctions redrive-execution reruns only FAILED/PENDING children, never succeeded items.

### 5. Security, cost, observability
- Least-privilege role: states:StartExecution on this state machine ARN (it starts children as itself), s3:ListBucket/GetObject on the source, s3:PutObject on the results prefix, lambda:InvokeFunction on the processor only, sqs:SendMessage on the DLQ only. Zero wildcards.
- CloudWatch Logs level ERROR, includeExecutionData false. One alarm on AWS/States ExecutionsFailed.
- Compute cost for [ITEM_COUNT] items: Express requests + GB-seconds, parent transitions, Lambda GB-seconds, S3 LIST/GET—show the math.

## Deliverables (emit every one)
1. state-machine.asl.json — complete Distributed Map definition.
2. trust-policy.json + execution-role-policy.json — every action on a named ARN.
3. dlq.json — SQS DLQ for failed item keys, 14-day retention.
4. deploy.sh — create-role, put-role-policy, create-state-machine --type STANDARD, start-execution on a 1,000-object test prefix.
5. "## Verification" — exact describe-map-run command, ItemCounts shape, manifest.json check.
6. "## Cost Estimate" table for [ITEM_COUNT] items with batch-size sensitivity.

## Evidence & Acceptance
Accepted only when: the test Map Run reaches SUCCEEDED with ItemCounts.Failed inside budget; manifest.json and per-status files exist; corrupting one test object fails only that item, never the run; the cost estimate matches the deployed config.

## Output Rules
- Imperative, production-ready, zero hedging. Real ASL fields only—verify uncertain names via MCP, never guess. Placeholders only where the user must substitute.
- Measurable bar: deployed and processing the 1,000-object test prefix in under 10 minutes.
```

## What You Get

- `state-machine.asl.json` — Standard-parent Distributed Map with `ItemReader`, `ItemBatcher`, failure-tolerance thresholds, `MaxConcurrency`, Lambda `Retry`/`Catch`, `ResultWriter`.
- `trust-policy.json` + `execution-role-policy.json` — every action on a named ARN.
- `dlq.json` — SQS dead-letter queue for failed item keys.
- `deploy.sh` — role creation, `create-state-machine --type STANDARD`, test run on a 1,000-object prefix.
- Verification (`describe-map-run`, `ItemCounts`, `redrive-execution`) and a 1M-item Cost Estimate table.

## Example Output

Excerpt:

    "ProcessImages": {
      "Type": "Map",
      "ItemProcessor": {
        "ProcessorConfig": { "Mode": "DISTRIBUTED", "ExecutionType": "EXPRESS" },
        "StartAt": "Transform",
        "States": { "Transform": { "Type": "Task",
          "Resource": "arn:aws:states:::lambda:invoke",
          "End": true } } },
      "ItemReader": { "Resource": "arn:aws:states:::s3:listObjectsV2",
        "Parameters": { "Bucket": "[SOURCE_BUCKET]", "Prefix": "[SOURCE_PREFIX]" } },
      "ItemBatcher": { "MaxItemsPerBatch": 100, "MaxInputBytesPerBatch": 262144 },
      "MaxConcurrency": 1000,
      "ToleratedFailurePercentage": 1,
      "ToleratedFailureCount": 1000,
      "ResultWriter": { "Resource": "arn:aws:states:::s3:putObject",
        "Parameters": { "Bucket": "[RESULTS_BUCKET]", "Prefix": "runs/" } },
      "End": true
    }

## AWS Services Used

- AWS Step Functions (Standard parent + Distributed Map; Express children)
- AWS Lambda (per-batch processor)
- Amazon S3 (ItemReader source; ResultWriter manifest)
- Amazon SQS (dead-letter queue)
- AWS IAM (least-privilege execution role)
- Amazon CloudWatch (ERROR logs, ExecutionsFailed alarm)

## Well-Architected Alignment

- Operational Excellence: per-status S3 manifest and `describe-map-run` `ItemCounts` to alarm on; failed runs recover via `redrive-execution`.
- Reliability: failure budgets isolate poison-pill items; Lambda `Retry` with backoff absorbs throttling; `Catch` routes failures to the SQS DLQ.
- Security: least-privilege IAM scoped to named bucket/function/queue/state-machine ARNs; logs exclude execution data.
- Performance Efficiency: `MaxConcurrency` capped at account Lambda concurrency; `ItemBatcher` packs 100 items per child (~100x fewer executions).
- Cost Optimization: batching, Express children, ERROR-only logging; a computed cost estimate is a required deliverable.

## Cost Notes

Prices us-east-1, June 2026. 1,000,000 items at 100/batch = 10,000 child executions.

- Express child requests: 10,000 x $1.00/1M = $0.01.
- Express duration: ~2s/batch at 512 MB: 10,000 x 2s x 0.5 GB x $0.00001667/GB-s = ~$0.17.
- Parent Standard transitions: one per child dispatch, 10,000 x $0.000025 = $0.25 (first 4,000/month free).
- Lambda: requests within the 1M/month free tier; compute 10,000 GB-s x $0.0000166667 = ~$0.17.
- S3: ItemReader LIST ~1,000 requests at $0.005/1,000—half a cent; processor GETs 1M x $0.0004/1,000 = $0.40.

Total: ~$0.43 of Step Functions orchestration, ~$1 all-in with Lambda and S3. The dominant lever is batch size: 10 items/batch means 100,000 children and ~10x the cost—tune `MaxItemsPerBatch` deliberately.

## Troubleshooting

- States.ExceedToleratedFailureThreshold fires immediately. Cause: thresholds left at the AWS default 0. Fix: set percentage 1 / count 1000, inspect the FAILED manifest, then `redrive-execution`.
- Every invoke throttles (Lambda.TooManyRequestsException). Cause: `MaxConcurrency` exceeds the account Lambda limit (default 1,000). Fix: lower it or request an increase.
- create-state-machine rejects the definition. Cause: the parent is Express—Distributed Map needs a Standard parent. Fix: `--type STANDARD`; children may stay EXPRESS.
- States.DataLimitExceeded on a child. Cause: batch payload exceeds 262,144 bytes. Fix: lower `MaxItemsPerBatch` or cap `MaxInputBytesPerBatch` at 262144.
- Items stuck pending, no manifest. Cause: role lacks `s3:PutObject` on results, `states:StartExecution` on its own ARN, or `lambda:InvokeFunction`. Fix: add the scoped action, update the role.
