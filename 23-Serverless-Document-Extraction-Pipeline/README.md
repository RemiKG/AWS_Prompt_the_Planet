# Serverless Document-Extraction Pipeline: Textract + Bedrock Structured Output with A2I Confidence Routing

Turn invoices, forms, and contracts into schema-valid JSON with Amazon Textract, Bedrock structured outputs, and A2I human review—so the 8% of pages your model is unsure about route to a person instead of silently poisoning your database.

## The Problem

You built a pipeline that calls Textract, pipes raw text into a Bedrock prompt that says "return JSON," and writes the result straight to DynamoDB. It works in the demo. In production it quietly fails: prose wraps the JSON 1 in 30 calls, an invoice total arrives as the string "$1,240.00" instead of the number 1240.00, a vendor name is hallucinated from a smudged scan, and nobody notices until finance finds 200 wrong rows at month-end. With no confidence gate, schema enforcement, or human in the loop, every low-quality extraction lands as if it were certain—and you cannot say which 200 rows, because per-field confidence was never captured. This requirements document forces the assistant to build it right: schema-valid JSON guaranteed by constrained decoding, a confidence threshold routing uncertain pages to Amazon A2I, and a computed cost-per-1,000-pages table before you deploy.

## Who This Is For

Startup engineers automating accounts-payable, KYC onboarding, claims intake, or lending document review who need extracted data trustworthy enough to act on automatically, with a measured human-review escape hatch and a known per-page cost.

## How to Use

1. Open your AI assistant (Kiro CLI, Claude Code, or Amazon Q Developer) in the repo where the pipeline will live.
2. Paste the entire System Prompt block below as your message.
3. Replace the placeholders: `[DOC_TYPE]` (e.g. invoice), `[CONFIDENCE_THRESHOLD]` (default 0.85), `[HOME_REGION]`, and `[MONTHLY_VOLUME]`.
4. Let the assistant verify Textract, Bedrock, and A2I availability in your region FIRST, then generate the Terraform, schema, and Lambda code.
5. Review the emitted Verification section, run `terraform plan`, then `terraform apply`.

Prerequisites:
- Required Access: an IAM principal that can create Lambda, Step Functions, S3, DynamoDB, IAM roles, Textract async jobs, Bedrock Converse, and SageMaker A2I FlowDefinitions. AdministratorAccess for the bootstrap; scope down after.
- Recommended Background: Terraform, Step Functions, JSON Schema (Draft 2020-12).
- Tools Required: the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`); AWS CLI v2; Terraform >= 1.6; Bedrock model access for an Anthropic Claude model that supports structured outputs.

Key Parameters: confidence threshold 0.85 (below routes to A2I), DetectDocumentText vs AnalyzeDocument FORMS+TABLES, Converse `outputConfig.textFormat` (type `json_schema`, Draft 2020-12 subset, `additionalProperties: false`), schema warm-up at deploy (grammar cached 24 h), 90-day Glacier Instant Retrieval lifecycle, DynamoDB PAY_PER_REQUEST, Lambda 1024 MB / 120 s, 3 retries plus SQS DLQ, CloudWatch retention 30 days.

Troubleshooting: If the assistant writes a Bedrock prompt that says "respond only with JSON" but does NOT set `outputConfig.textFormat`, that is the failure to catch—prompt-based JSON is best-effort and leaks prose under load; constrained decoding is the only contract that holds.

## System Prompt

```
# Serverless Document-Extraction Pipeline — Architecture & Implementation Request

## Project Overview
Design and generate a production-ready, serverless pipeline that extracts structured data from [DOC_TYPE] documents (PDF/PNG/JPEG/TIFF) landing in S3, enforces a strict JSON schema at generation time, and routes low-confidence results to Amazon A2I human review. Home region: [HOME_REGION]. Volume: [MONTHLY_VOLUME]. Nothing reaches DynamoDB unless it is schema-valid AND above the confidence threshold or human-approved.

## Verify Reality FIRST (before writing any code)
1. With the AWS Documentation MCP tools (aws_read_documentation, aws_search_documentation), CONFIRM Textract, Bedrock, and A2I FlowDefinitions are all available in [HOME_REGION]; if not, STOP and report the nearest region with all three.
2. CONFIRM the Converse structured-output request shape — outputConfig.textFormat with type json_schema (Draft 2020-12 subset) — and which model IDs currently support it in [HOME_REGION]. Never assume a model ID; look it up.
3. CONFIRM the schema subset: additionalProperties must be false; minimum/maximum, minLength/maxLength, and recursive $ref are NOT supported — move bounds checks into the Lambda guard.
4. CONFIRM the Textract async APIs (StartDocumentAnalysis/GetDocumentAnalysis vs StartDocumentTextDetection) and that blocks carry Confidence 0-100.
5. Report each as "Verified: <fact> — <source>". If unverifiable, use the safe generic form and flag it; NEVER invent an API name, model ID, quota, or price.

## Detailed Requirements

### 1. Core Functionality
- s3:ObjectCreated on the raw bucket -> EventBridge -> Step Functions (Standard).
- Textract: async StartDocumentAnalysis with FORMS and TABLES; poll GetDocumentAnalysis; persist per-field Confidence (0-100).
- Bedrock: send Textract text and key-value pairs to the verified Claude model via Converse with outputConfig.textFormat set to my schema — constrained decoding GUARANTEES schema-valid JSON. NEVER rely on "return only JSON" prompt text.
- Routing: min-confidence across required fields (Confidence/100). At or above [CONFIDENCE_THRESHOLD] (default 0.85): conditional-put to DynamoDB, STATUS=auto_approved; below: StartHumanLoop on the A2I FlowDefinition, STATUS=pending_review.
- A2I callback: on completion, the corrected JSON overwrites the record, STATUS=human_approved.

### 2. The JSON Schema (deliverable)
schema/[DOC_TYPE].schema.json — totals as number never string, dates as ISO-8601 "date" strings, currency code as enum, "required" populated, "additionalProperties": false. Enforce twice: outputConfig.textFormat at generation, jsonschema Lambda guard before any write.

### 3. Performance & Reliability
- A new schema triggers grammar compilation (up to minutes), cached 24 hours — warm it with one Converse call at deploy.
- Step Functions: 3 retries, exponential backoff, on ThrottlingException; exhausted retries land in an SQS DLQ for replay.
- Lambda: Python 3.12, 1024 MB, 120 s. Idempotency keyed on the S3 object ETag via conditional put so redelivery never double-extracts.

### 4. Cost Targets
COMPUTE and SHOW a cost-per-1,000-pages table: Textract (DetectDocumentText $1.50 vs AnalyzeDocument FORMS+TABLES $65 per 1,000 pages in us-east-1; confirm [HOME_REGION] pricing), Bedrock tokens for the verified model with stated per-page assumptions, A2I at $0.03 per reviewed page (private workforce), plus Lambda/Step Functions/DynamoDB. State the review rate assumed (default ~8% below 0.85) and recommend the cheaper Textract path whenever plain text can fill the schema.

### 5. Security & Compliance
- S3: SSE-KMS customer-managed key, all BlockPublicAccess flags, aws:SecureTransport deny policy, versioning, 90-day Glacier Instant Retrieval lifecycle.
- IAM: one least-privilege role per Lambda; no Resource: "*" on textract:, bedrock:, or sagemaker: actions.
- CloudWatch Logs: 30-day retention, KMS-encrypted.

## Deliverables Requested (produce all)
1. Terraform (>= 1.6): both buckets, DynamoDB (PAY_PER_REQUEST), state machine, five Lambdas, EventBridge rules, SQS DLQ, KMS key, A2I FlowDefinition with worker UI template, scoped IAM roles.
2. schema/[DOC_TYPE].schema.json inside the verified subset.
3. Python 3.12 handlers: Textract poller, Bedrock caller, confidence router, validation guard, A2I callback.
4. cost.md — the cost-per-1,000-pages table, every assumption stated.
5. README.md — deploy steps, schema warm-up, confidence-tuning guidance.

## Verification / Acceptance (emit with evidence)
- terraform validate output and a terraform plan resource summary.
- One round trip producing schema-valid JSON, and one sample below [CONFIDENCE_THRESHOLD] creating a human loop.
- Three reviewer-runnable assertions: no IAM policy contains Resource: "*"; the Converse request sets outputConfig.textFormat; the raw bucket denies non-TLS requests.

## Error Management
- Bedrock 400 on the schema -> unsupported subset feature; reshape it, never relax additionalProperties.
- Throttling -> retry with backoff, then DLQ; never write partial JSON.
- Textract job FAILED -> STATUS=extraction_failed, to DLQ, never the processed table.
- Guard rejects a record -> route to A2I, never auto-approve.
- A2I loop creation fails -> keep STATUS=pending_review, set an error marker, alarm on it; never drop the document.

Use Terraform. Optimize for the lowest cost-per-1,000-pages that still enforces the schema. Output the files and Verification section without preamble. The pipeline must be production-ready and deployable in under 30 minutes.
```

## What You Get

- `main.tf`, `variables.tf`, `outputs.tf` — both S3 buckets, DynamoDB (PAY_PER_REQUEST), state machine, five Lambdas, EventBridge rules, SQS DLQ, KMS key, one least-privilege IAM role per function.
- `a2i.tf` — the A2I FlowDefinition plus the worker task UI template.
- `schema/[DOC_TYPE].schema.json` — typed, required-enforced, additionalProperties false, inside the structured-output subset.
- `src/` — Python 3.12 handlers: Textract poller, Bedrock caller, confidence router, validation guard, A2I callback.
- `cost.md` — the cost-per-1,000-pages table with stated assumptions.
- `README.md` — deploy steps, schema warm-up, confidence tuning, verification assertions.

## Example Output

Verified: Textract, Bedrock, and A2I FlowDefinitions available in us-east-1 — aws_read_documentation.
Verified: Converse structured outputs use outputConfig.textFormat (type json_schema, Draft 2020-12 subset); supporting Claude model ID confirmed.
Verified: additionalProperties must be false; minLength/maximum unsupported — bounds moved to the Lambda guard.

Cost per 1,000 pages (invoice, FORMS+TABLES, ~8% to review):
- Textract FORMS ($50/1k) + TABLES ($15/1k) = $65.00
- Bedrock Claude Sonnet-class, 1,500 in / 400 out tokens/page at $3/$15 per million = $10.50
- A2I: 80 pages x $0.03 = $2.40
- Lambda + Step Functions + DynamoDB = ~$1.20
- Total: ~$79.10. Text-only docs on DetectDocumentText ($1.50/1k): ~$15.60.

## AWS Services Used

Amazon Textract, Amazon Bedrock (Converse API, structured outputs), Amazon Augmented AI (A2I), Amazon S3, AWS Step Functions, AWS Lambda, Amazon DynamoDB, Amazon EventBridge, Amazon SQS, AWS KMS, Amazon CloudWatch, AWS IAM.

## Well-Architected Alignment

- Operational Excellence: per-state retries, SQS DLQ replay, 30-day CloudWatch logs, an evidence-backed verification section before deploy.
- Security: SSE-KMS customer-managed key, BlockPublicAccess, TLS-only bucket policy, least-privilege role per Lambda, no wildcard Textract/Bedrock/SageMaker actions.
- Reliability: ETag-keyed idempotent writes, 3-attempt backoff on ThrottlingException, a guard that blocks partial writes, human-review fallback so uncertain data never auto-approves.
- Performance Efficiency: async Textract jobs, constrained decoding (no retry-on-bad-JSON loop), grammar caching warmed at deploy, 1024 MB Lambdas.
- Cost Optimization: computed cost-per-1,000-pages, cheaper Textract path when the schema allows, PAY_PER_REQUEST DynamoDB, Glacier Instant Retrieval at 90 days, A2I billed only on the ~8% routed.

## Cost Notes

Real US East (N. Virginia) figures: Textract DetectDocumentText is $1.50 per 1,000 pages; AnalyzeDocument FORMS is $50 and TABLES $15 per 1,000. A2I private-workforce review is $0.03 per reviewed page for the first 100,000 objects each month, $0.02 thereafter. At 50,000 invoice pages/month on FORMS+TABLES with an 8% review rate: ~$3,250 Textract, ~$525 Bedrock (Sonnet-class, 1,500 in / 400 out tokens per page), $120 A2I, under $60 in Lambda/Step Functions/DynamoDB — roughly $0.079 per page all-in. Text-only documents on DetectDocumentText collapse Textract to $75/month and the total to ~$780.

## Troubleshooting

- JSON contains prose or wrong types. Cause: a "return only JSON" instruction instead of the structured-output contract. Fix: set `outputConfig.textFormat` so constrained decoding enforces the schema at generation time.
- Bedrock rejects the schema with a 400. Cause: the subset forbids minimum/maximum, minLength/maxLength, recursive $ref, and any additionalProperties except false. Fix: reshape the schema; enforce bounds in the Lambda guard.
- First extraction after deploy is slow. Cause: grammar compilation on first use (up to minutes), cached 24 hours. Fix: warm the schema with one Converse call at deploy.
- Everything auto-approves. Cause: confidence read from Bedrock, not Textract. Fix: gate on min Textract Confidence (0-100, divide by 100) across required fields.
- A2I loops never start. Cause: missing FlowDefinition/worker UI, or no sagemaker:StartHumanLoop. Fix: deploy `a2i.tf`; grant the router Lambda role StartHumanLoop.
- Duplicate records. Cause: EventBridge redelivery without idempotency. Fix: key writes on the S3 object ETag with a conditional put.
- Throttling drops documents. Cause: ThrottlingException with no retry. Fix: 3-attempt backoff, then the SQS DLQ for replay.
