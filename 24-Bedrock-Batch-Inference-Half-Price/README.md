# Bedrock Batch Inference at Half Price: JSONL Job Builder for Embeddings & Bulk Generation at Scale

Generate the S3 layout, JSONL input files, IAM trust policy, and `create-model-invocation-job` call that run your bulk LLM workload at 50% of on-demand pricing—so you stop paying real-time rates for jobs that don't need real-time answers.

## The Problem

Most teams send every Bedrock request through the synchronous `InvokeModel` / `Converse` APIs and pay full on-demand rates—even when no human is waiting. Embedding 1,000,000 support tickets with Titan Text Embeddings V2 at the on-demand $0.02 per 1M input tokens (~512 tokens/ticket = ~512M tokens) costs $10.24; the same job run as **batch inference** bills at the 50% batch rate published on the Amazon Bedrock pricing page = $5.12. Scale that to nightly re-embedding or summarizing 200,000 transcripts and the on-demand premium becomes a five-figure line item.

The discount is real but the on-ramp is fiddly: batch demands a specific **JSONL** record shape (`recordId` + `modelInput`), separate S3 input and output prefixes, an IAM role with `bedrock.amazonaws.com` as trust principal, and files inside the 1 GB per-file cap and per-job size quota. A wrong `modelInput` body fails the whole job at validation; a reused synchronous-API role dies with `AccessDeniedException`. Teams either never adopt batch (and overpay 2x) or burn a day hand-assembling JSONL.

This prompt makes an AI assistant produce a complete, production-ready batch package—correct JSONL, least-privilege role, the exact CLI call, EventBridge monitoring, and a real-dollar cost estimate—after first verifying your model, region, and quotas actually support it.

## Who This Is For

- Startups running **embedding pipelines** (RAG ingestion, semantic-search backfill) over large corpora.
- Teams doing **bulk generation**: summarization, classification, extraction, translation across tens of thousands of records where latency is irrelevant.
- Cost owners who suspect non-urgent GenAI spend sits on on-demand rates and want the 50% cut without learning the batch API.

## How to Use

1. Open Kiro CLI, Claude Code, or any AI assistant with the AWS Documentation MCP server and AWS CLI access.
2. Paste the System Prompt below into a new session.
3. Replace the bracketed placeholders: `[MODEL_ID]`, `[REGION]`, `[RECORD_COUNT]`, `[TOKENS_PER_RECORD]`, `[S3_BUCKET]`, `[ACCOUNT_ID]`, and the workload description.
4. Let the assistant verify batch eligibility and current quotas, then review the JSONL sample, IAM role JSON, and CLI command before running anything.
5. Apply the role, upload the JSONL to the input prefix, and run the printed `aws bedrock create-model-invocation-job` command.

Prerequisites:
- Required Access: an IAM principal allowed to call `bedrock:CreateModelInvocationJob`, `bedrock:GetModelInvocationJob`, `iam:CreateRole` / `iam:PassRole`, and S3 read/write on the target bucket; model access for `[MODEL_ID]` granted on the Bedrock console Model access page.
- Recommended Background: S3 prefixes, JSON, reading job-status output.
- Tools Required: AWS CLI v2; AWS Documentation MCP server tools `aws___search_documentation` and `aws___read_documentation` to confirm the `modelInput` schema and current batch quotas before generating.

Key Parameters: model (`amazon.titan-embed-text-v2:0` default), region (`us-east-1`), input prefix (`s3://[S3_BUCKET]/batch-input/`), output prefix (`s3://[S3_BUCKET]/batch-output/`), per-file cap (1 GB, not adjustable), job-size quota (5 GB most models; 100 GB Amazon Nova Lite/Pro), minimum records per job (100 for most models), roleArn (`BedrockBatchInferenceRole`), jobName (`embed-backfill-YYYYMMDD`), timeoutDurationInHours (72).

Troubleshooting: If `create-model-invocation-job` returns `AccessDeniedException`, it is expected the first time—the role's trust policy must allow `bedrock.amazonaws.com` to assume it AND grant S3 access on the exact prefixes; a role that works for synchronous `InvokeModel` is NOT sufficient because batch runs as the service, not as you.

## System Prompt

```
# Amazon Bedrock Batch Inference Job — Build Request

You are a senior AWS GenAI cost-optimization engineer. Produce a complete, production-ready Amazon Bedrock batch inference package that runs my bulk workload at the 50% batch discount the Bedrock pricing page documents versus on-demand. Output runnable artifacts, not explanations of what batch is.

## My Workload
- Model ID: [MODEL_ID]  (e.g. amazon.titan-embed-text-v2:0, anthropic.claude-3-5-sonnet-20240620-v1:0)
- Region: [REGION]
- Task: [DESCRIBE: embed / summarize / classify / extract over what data]
- Record count: [RECORD_COUNT]; average tokens per record: [TOKENS_PER_RECORD]
- S3 bucket: [S3_BUCKET]; AWS account ID: [ACCOUNT_ID]

## Verify Reality FIRST — before generating anything
1. Confirm [MODEL_ID] supports batch inference in [REGION] via aws___search_documentation and aws___read_documentation against the "Supported Regions and models for batch inference" page; do NOT assume. If not eligible, STOP and name the nearest supported region or model.
2. Confirm the EXACT modelInput body for [MODEL_ID] from its model-parameters page. Titan Text Embeddings V2 takes {"inputText": "..."}; Claude takes {"anthropic_version":"bedrock-2023-05-31","max_tokens":N,"messages":[...]}. A wrong shape fails the whole job at validation.
3. Confirm fit: batch processes each record independently and does NOT support tool calling or structured output. If my task needs either, STOP.
4. Read the CURRENT quotas: "Minimum number of records per batch inference job" (100 for most models), "Records per input file per batch inference job", "Batch inference input file size (in GB)" (1 GB, not adjustable), "Batch inference job size (in GB)" (5 GB most models; 100 GB Amazon Nova Lite/Pro). Report the values. If [RECORD_COUNT] is below the minimum, STOP—batch will reject the job.
5. List anything unverified under ASSUMPTIONS. Never invent a quota, price, or API field; if not 95% sure, write the safe generic form and tell me to confirm in the console.

## Deliverables — produce ALL of these
1. S3 layout: input prefix s3://[S3_BUCKET]/batch-input/ and a SEPARATE output prefix s3://[S3_BUCKET]/batch-output/.
2. A sample input.jsonl: 3 lines, one JSON object per line, exactly { "recordId": "REC0000001", "modelInput": { ...schema for [MODEL_ID]... } }, recordId unique per line.
3. A file-splitting rule: shard into input-000.jsonl, input-001.jsonl ... when records exceed the per-file quota or 1 GB, staying under the job-size quota.
4. A least-privilege IAM role BedrockBatchInferenceRole as JSON: trust policy with Principal Service bedrock.amazonaws.com plus aws:SourceAccount = [ACCOUNT_ID] and aws:SourceArn = arn:aws:bedrock:[REGION]:[ACCOUNT_ID]:model-invocation-job/* (confused-deputy guard); permissions granting only s3:GetObject + s3:ListBucket on the input side and s3:PutObject on the output prefix.
5. The exact CLI call: aws bedrock create-model-invocation-job --model-id [MODEL_ID] --job-name embed-backfill-YYYYMMDD --role-arn <roleArn> --input-data-config '{"s3InputDataConfig":{"s3Uri":"s3://[S3_BUCKET]/batch-input/"}}' --output-data-config '{"s3OutputDataConfig":{"s3Uri":"s3://[S3_BUCKET]/batch-output/"}}' --timeout-duration-in-hours 72 --region [REGION].
6. Monitoring: aws bedrock get-model-invocation-job --job-identifier <jobArn>, how to read Submitted / InProgress / Completed / PartiallyCompleted / Failed / Expired, plus an EventBridge rule for Bedrock job state changes so production runs notify instead of polling.
7. A cost table: on-demand vs batch (50% of on-demand) for [RECORD_COUNT] x [TOKENS_PER_RECORD], citing the per-1M-token rates used; show the dollar saving per pass.
8. A decision note: batch wins for non-urgent, high-volume, bursty jobs—no commitment, 50% off, results in hours, and batch is not even supported on provisioned models; Provisioned Throughput wins only for guaranteed low-latency sustained capacity on 1- or 6-month commitments; on-demand wins for interactive requests.

## Verification & Acceptance — emit evidence it works
- A local dry-validation one-liner: json.loads every line, assert recordId unique and modelInput matches the schema, before any upload.
- The output layout to check after completion: per-input-file .jsonl.out results plus manifest.json.out with record and token counts, in a job-specific subfolder of the output prefix.
- A Verification checklist with [ ] items: model batch-eligible in [REGION]; modelInput schema confirmed; [RECORD_COUNT] >= minimum-records quota; every file <= 1 GB and job under its size quota; output prefix != input prefix; trust policy carries SourceAccount + SourceArn.

## Error Handling — Cause -> Fix for each
- AccessDeniedException at submission -> trust policy missing bedrock.amazonaws.com or S3 permissions miss the exact prefixes; a synchronous InvokeModel role is NOT sufficient.
- ValidationException -> modelInput shape wrong for [MODEL_ID].
- Job rejected on size -> below the minimum-records quota, a file over 1 GB, or the job over its size quota.
- ResourceNotFoundException -> model access not granted in [REGION].
- Status Expired -> the job outran timeout-duration-in-hours; check manifest.json.out for what completed, resubmit the remainder.

## Constraints
- Cost-conscious by default: never recommend Provisioned Throughput unless I state a sustained low-latency SLA.
- Imperative, zero hedging. Artifacts in fenced jsonl / json / bash blocks, no preamble.
- The package must be production-ready and runnable in under 10 minutes after I paste my values. End with the Verification checklist, then one line stating the dollar saving versus on-demand.
```

## What You Get

- S3 layout with separate `batch-input/` and `batch-output/` prefixes.
- `input.jsonl` — a 3-line, model-correct sample in the `recordId` + `modelInput` shape, plus a dry-validation one-liner.
- File-splitting rule under the 1 GB per-file cap and per-job size quota.
- `BedrockBatchInferenceRole` — trust policy with `aws:SourceAccount` / `aws:SourceArn` confused-deputy guards plus least-privilege S3 permissions, as JSON.
- The `aws bedrock create-model-invocation-job` command with `s3InputDataConfig` / `s3OutputDataConfig` populated.
- `get-model-invocation-job` polling plus an EventBridge rule for job state changes.
- A cost-estimate table (on-demand vs batch at 50% off) and a batch vs Provisioned Throughput vs on-demand decision note.
- A Verification checklist with concrete `[ ]` acceptance items.

## Example Output

ASSUMPTIONS: amazon.titan-embed-text-v2:0 confirmed batch-eligible in us-east-1 via docs; ~512 tokens/record assumed; quotas read from Service Quotas (minimum 100 records/job, 1 GB/file, 5 GB/job).

input.jsonl (sample):
{ "recordId": "REC0000001", "modelInput": { "inputText": "Customer cannot reset password after MFA enrollment." } }

Cost estimate (1,000,000 records x ~512 tokens = ~512M input tokens, Titan Text Embeddings V2):
On-demand @ $0.02 / 1M tokens = $10.24. Batch @ 50% = $5.12. Saving: $5.12 per full pass, every pass.

Verification checklist:
[ ] Model batch-eligible in us-east-1  [ ] modelInput = {"inputText": ...}  [ ] 1,000,000 records >= 100 minimum  [ ] output prefix != input prefix  [ ] trust = bedrock.amazonaws.com + SourceAccount/SourceArn.

## AWS Services Used

Amazon Bedrock (batch inference / `CreateModelInvocationJob`), Amazon S3, AWS IAM, AWS Service Quotas, Amazon EventBridge (job state-change notifications), Amazon CloudWatch, AWS CLI v2.

## Well-Architected Alignment

- **Cost Optimization** — batch bills at 50% of on-demand per the Bedrock pricing page; the prompt forbids recommending Provisioned Throughput without a sustained low-latency SLA and emits the dollar saving as a deliverable.
- **Security** — least-privilege `BedrockBatchInferenceRole` scoped to exact S3 prefixes; trust policy carries `aws:SourceAccount` and `aws:SourceArn` to defeat the confused-deputy problem.
- **Operational Excellence** — verify-reality-first (model/region/quota/tool-support checks), runnable CLI, EventBridge notifications instead of blind polling, `[ ]` acceptance checklist.
- **Reliability** — file-splitting respects the 1 GB per-file cap and per-job quota so jobs don't fail at submission; Cause -> Fix coverage for the five most common failures including `Expired` timeouts.
- **Performance Efficiency** — routes non-urgent, high-volume work to async batch; reserves synchronous APIs for latency-sensitive requests.

## Cost Notes

The Amazon Bedrock pricing page lists batch inference at **50% of on-demand rates** for supported models, charged per token with no commitment. At published rates: Claude 3.5 Sonnet on-demand input $3.00 / 1M tokens drops to $1.50 batch, output $15.00 drops to $7.50; Titan Text Embeddings V2 $0.02 / 1M drops to $0.01. No provisioning fee, no idle cost, no Provisioned Throughput commitment ($/model-unit/hour on 1- or 6-month terms) to amortize—you pay only for tokens at half rate. The only adjacent cost is S3 storage for the JSONL (S3 Standard ~$0.023/GB-month), negligible at these volumes.

## Troubleshooting

- **`AccessDeniedException` on the service role** — Cause: trust policy doesn't allow `bedrock.amazonaws.com`, or S3 permissions miss the exact prefixes. Fix: use the generated `BedrockBatchInferenceRole` JSON verbatim; a synchronous `InvokeModel` role won't work because batch runs as the service.
- **`ValidationException` on `modelInput`** — Cause: body doesn't match the schema for `[MODEL_ID]`. Fix: confirm the exact body on the model-parameters docs page; Titan Text Embeddings V2 = `{"inputText": "..."}`.
- **Job rejected at submission for size** — Cause: below the "Minimum number of records per batch inference job" quota (100 for most models), a file over 1 GB, or the job over its size quota (5 GB for most models). Fix: check Service Quotas and shard accordingly.
- **`ResourceNotFoundException`** — Cause: model access not granted on the Bedrock Model access page in `[REGION]`. Fix: request access and wait for "Access granted".
- **Results not where expected** — Cause: outputs land under the output prefix in a job-specific subfolder as `.jsonl.out` files plus `manifest.json.out` with record and token counts. Fix: keep `batch-output/` distinct from `batch-input/`; read the manifest first—`PartiallyCompleted` means some records failed and carry error detail in their output lines.
