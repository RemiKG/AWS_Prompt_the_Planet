# Rekognition UGC Moderation Pipeline: Calibrated Block/Review/Allow Thresholds That Keep Human Review Under 4%

Event-driven S3 + EventBridge + Rekognition image moderation with a tiered confidence policy across all 10 moderation categories, an SQS human-review queue, quarantine-with-appeal, and honest false-positive math—so unsafe images never go live and moderators only see the 3–4% that genuinely need human eyes.

## The Problem

Your platform accepts user-uploaded images. At 100,000 uploads/day, reviewing everything by hand costs 100,000 × 12 seconds = 333 reviewer-hours per day — roughly 42 full-time moderators before you've shipped a single feature. So most teams do one of two bad things instead: they serve uploads instantly and delete later (unsafe content is live for minutes, advertisers and Apple's App Review Guideline 1.2 both notice), or they auto-block everything Amazon Rekognition flags at its default 50 confidence — which nukes swimsuit photos, wine-at-dinner shots, breastfeeding pictures, and classical art, and fills support with furious legitimate users.

The teams that get this right tier the decision: auto-block only what the model scores ≥95 in severe categories, auto-allow the clean ~95%, and route the ambiguous middle band to a small human queue. The hard parts are (1) the policy table — which of Rekognition's 10 top-level moderation categories map to which action at which confidence — and (2) the math nobody runs: what each threshold point costs in reviewer-hours and dollars. This prompt makes an AI assistant produce both, plus the production-ready pipeline around them.

## Who This Is For

- Social, community, gaming, and marketplace startups with image UGC and no trust-and-safety headcount yet
- Dating apps that must block explicit content before a photo reaches anyone's feed
- Founders facing Apple App Review Guideline 1.2 (UGC apps require filtering, reporting, and blocking) before launch
- Anyone who tried Rekognition moderation, saw false positives, and gave up on automation entirely

## How to Use

1. Open Kiro CLI, Claude Code, or Amazon Q Developer in the repository where your infrastructure code lives.
2. Copy the System Prompt below and paste it as your request (or save it as `.kiro/steering/moderation.md`).
3. Replace the bracketed placeholders: `[UPLOADS_BUCKET]`, `[REGION]`, `[DAILY_UPLOAD_VOLUME]`, `[PLATFORM_TYPE]`, `[TARGET_REVIEW_RATE]`.
4. Let the assistant complete its verification pass (Rekognition region availability, bucket notification config, TPS quota) before accepting any generated code.
5. Review the generated policy table with whoever owns your content policy — it IS your content policy — then apply the Terraform.
6. Run `calibrate.py` against 1,000 real images from your platform and tune thresholds in SSM Parameter Store, no redeploy needed.
7. Upload one clean and one flagged test image and confirm the acceptance checks pass.

**Prerequisites**

- **Required Access:** AWS account with permissions to create IAM roles, Lambda functions, EventBridge rules, SQS queues, and DynamoDB tables; `rekognition:DetectModerationLabels`; an S3 uploads bucket (existing or created by the IaC).
- **Recommended Background:** Basic Terraform and S3 event flows; a written content policy (even one paragraph) for your platform.
- **Tools Required:** Kiro CLI or Claude Code with the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) for region and quota verification; AWS CLI v2; Terraform >= 1.6; Python 3.13.

**Key Parameters:** MinConfidence (50), severe auto-block threshold (>=95), contextual review threshold (>=75), quarantine retention (30 days), SQS visibility timeout (1h, maxReceiveCount 3 -> DLQ), Lambda (512 MB / 30s / reserved concurrency 40), overturn-rate retune trigger (40%), target review rate (<=4%).

**Troubleshooting:** If the first calibration run shows a 10%+ flag rate, that's expected on lifestyle- or fitness-heavy platforms — Swimwear or Underwear and Alcohol fire constantly at 50–70 confidence. Raise the contextual review threshold before assuming the model is broken.

## System Prompt

```
# UGC Image Moderation Pipeline — Architecture & Policy Design Request

## Project Overview

Design and generate a production-ready, event-driven image moderation pipeline for a [PLATFORM_TYPE: general-audience social / dating / marketplace / kids] platform handling [DAILY_UPLOAD_VOLUME: 100,000] uploads/day in [REGION: us-east-1]. Uploads land in s3://[UPLOADS_BUCKET]/uploads/, Amazon Rekognition DetectModerationLabels scores every image, and a tiered confidence policy decides BLOCK, REVIEW, or ALLOW. Nothing is publicly served before a verdict. Target human-review rate: <=[TARGET_REVIEW_RATE: 4]% of uploads.

## Verify Before Generating

1. Confirm Amazon Rekognition is available in [REGION] using the AWS Documentation MCP server (aws_read_documentation). If it is not, propose the nearest supported region and quantify the cross-region data-transfer cost.
2. Run `aws s3api get-bucket-notification-configuration --bucket [UPLOADS_BUCKET]`. If EventBridge notifications are not enabled, include enabling them in the IaC.
3. Check the account's DetectModerationLabels TPS quota in Service Quotas and set Lambda reserved concurrency below it (e.g., 40 if the quota is 50).
4. Call DetectModerationLabels once on a test image and record the ModerationModelVersion returned; pin it into every verdict record.
Never reference a resource, API field, or quota you have not confirmed. If a check fails, stop and report it instead of generating around it.

## Detailed Requirements

### 1. Threshold Policy — the core artifact
Produce a policy table covering all 10 top-level Rekognition moderation categories (Explicit; Non-Explicit Nudity of Intimate Parts and Kissing; Swimwear or Underwear; Violence; Visually Disturbing; Drugs & Tobacco; Alcohol; Rude Gestures; Gambling; Hate Symbols), each mapped to BLOCK / REVIEW / ALLOW with explicit confidence bands. General-audience defaults:
- BLOCK at confidence >=95: Explicit, Hate Symbols, Visually Disturbing, graphic Violence. Move the object to s3://[UPLOADS_BUCKET]/quarantine/ — never delete immediately; apply a 30-day lifecycle expiry to preserve an appeals window.
- REVIEW: 50–94.9 in those four severe categories; >=75 in contextual categories (Swimwear or Underwear, Non-Explicit Nudity, Drugs & Tobacco, Alcohol, Gambling, Rude Gestures). Exception: every Self-Harm detection at any returned confidence routes to a priority queue and triggers the user-safety runbook — never a silent block.
- ALLOW: no labels returned at MinConfidence=50, or contextual labels below the review band (tag-and-allow — e.g., Alcohol at 62 on a dinner photo).
Store the table as JSON in SSM Parameter Store at /moderation/policy so thresholds tune without a redeploy. Adapt defaults to [PLATFORM_TYPE] (kids platform: Swimwear or Underwear becomes REVIEW >=50).

### 2. Pipeline Architecture
S3 (Block Public Access ON) -> EventBridge rule on "Object Created" filtered to the uploads/ prefix -> Lambda (Python 3.13, 512 MB, 30s timeout) -> DetectModerationLabels (MinConfidence=50, S3Object reference; JPEG/PNG only, 15 MB max) -> conditional DynamoDB write (attribute_not_exists on the object key, so duplicate EventBridge deliveries stay idempotent) -> action: copy to approved/, move to quarantine/, or send to the SQS review queue (SSE enabled, 1-hour visibility timeout, maxReceiveCount 3, redrive to DLQ). FAIL CLOSED: on any Rekognition error the image stays unpublished and the event goes to a Lambda destination DLQ for replay. HEIC/WebP uploads must be transcoded to JPEG before moderation or rejected — never auto-allowed.

### 3. Human-Review Routing Math — show your work
Compute and print: at [DAILY_UPLOAD_VOLUME] uploads/day with a 6.5% calibrated flag rate, the defaults yield ~1.8% auto-block, ~3.2% review, ~95% auto-allow. 3,200 reviews × 12 s/decision = 10.7 reviewer-hours/day (~1.5 FTE). State explicitly that each +1 point of review rate costs ~3.3 labor-hours per 100k images. Offer Amazon A2I (flow definition + private workforce, $0.03/object for the first 100,000 objects/month) as the managed alternative to the SQS queue; recommend SQS when an internal review tool already exists or volume exceeds A2I economics.

### 4. False-Positive Honesty
Confidence measures the model's certainty the label applies — not the probability my policy was violated. Swimwear, breastfeeding, classical art, medical imagery, and toy weapons all false-flag. Requirements: log every reviewer decision; compute weekly per-category overturn rates; if a category's overturn rate exceeds 40%, raise its review threshold 5 points. Include a pre-launch calibration script that scores 1,000 real images from my platform and prints per-category confidence histograms. State plainly in the runbook that Rekognition is not a CSAM detector — that requires dedicated hash-matching tooling and a mandatory-reporting workflow outside this pipeline's scope.

### 5. Cost Ceiling
Automated path <=$110 per 100,000 images in us-east-1: Rekognition at $0.001/image (tier 1) = $100 dominates; Lambda ~$1; S3-to-EventBridge events at $1.00/million = $0.10; DynamoDB on-demand + SQS + S3 requests < $2. Output the full cost table including the human-review labor line — thresholds, not instance types, are the cost lever here.

### 6. Observability
CloudWatch custom metrics: FlagRate, ReviewQueueDepth, OverturnRate. Alarms: ReviewQueueDepth > 500 for 15 minutes; FlagRate > 2× the 7-day baseline (raid in progress or a silent model update); ModerationModelVersion change -> trigger recalibration.

## Deliverables

1. ASCII architecture diagram
2. Terraform for every resource with least-privilege IAM (Lambda role: rekognition:DetectModerationLabels, s3:GetObject on uploads/*, s3:PutObject on approved/* and quarantine/*, dynamodb:PutItem, sqs:SendMessage — nothing broader)
3. moderation_handler.py with the policy engine reading /moderation/policy
4. The threshold policy JSON
5. calibrate.py (batch-scores a sample prefix, prints per-category confidence histograms and predicted block/review/allow split)
6. REVIEW_RUNBOOK.md (claim -> decide -> record -> overturn logging -> appeal path)
7. Cost table per 100,000 images

## Acceptance Evidence

Prove it works before declaring done: upload one clean and one flagged test image; show the clean object in approved/ and the flagged one quarantined or queued; show an anonymous GET on a quarantine/ object returns 403; show both DynamoDB verdict items with ModerationModelVersion populated. Align with the Well-Architected Framework: Security (default-deny serving, least privilege, SSE everywhere), Reliability (fail closed, DLQ replay), Cost Optimization (review rate as the dominant cost driver). The pipeline must deploy in under 15 minutes. Output the Terraform first, then code, then runbook — no preamble.
```

## What You Get

1. **Architecture diagram** — the full S3 → EventBridge → Lambda → Rekognition → {approved/ | quarantine/ | SQS} flow
2. **Terraform** — bucket policies (Block Public Access, prefix isolation), EventBridge rule, Lambda + reserved concurrency, SQS review queue + DLQ, DynamoDB verdict table, CloudWatch alarms, least-privilege IAM
3. **`moderation_handler.py`** — policy engine with fail-closed error handling and idempotent verdict writes
4. **`/moderation/policy` JSON** — the 10-category threshold table, tunable in Parameter Store without redeploys
5. **`calibrate.py`** — scores 1,000 of your real images, prints confidence histograms and the predicted block/review/allow split
6. **`REVIEW_RUNBOOK.md`** — reviewer workflow with overturn logging and the appeal path
7. **Cost table** — automated vs. human-review cost per 100k images, with the per-threshold-point labor math

## Example Output

```
MODERATION POLICY — general-audience defaults (ModerationModelVersion 7.0)
Explicit ............. BLOCK >=95 | REVIEW 50–94.9
Hate Symbols ......... BLOCK >=95 | REVIEW 50–94.9
Violence ............. BLOCK >=95 (graphic) | Self-Harm: ALWAYS priority review
Swimwear/Underwear ... REVIEW >=75 | tag-and-ALLOW below
Alcohol .............. REVIEW >=75 | tag-and-ALLOW below

ROUTING FORECAST @ 100,000 uploads/day (6.5% calibrated flag rate):
  auto-block 1,800 | human review 3,200 (10.7 reviewer-hours, ~1.5 FTE) | auto-allow 95,000

VERDICT: {"key":"uploads/u8123/img_4421.jpg","verdict":"REVIEW",
 "top_label":"Swimwear or Underwear","confidence":81.4,"model":"7.0",
 "queued":"moderation-review-queue"}
```

## AWS Services Used

Amazon Rekognition, Amazon S3, Amazon EventBridge, AWS Lambda, Amazon SQS, Amazon DynamoDB, AWS Systems Manager Parameter Store, Amazon CloudWatch, Amazon Augmented AI (optional), AWS IAM

## Well-Architected Alignment

- **Security:** Default-deny serving — no object is publicly reachable until a verdict; Block Public Access on; quarantine prefix returns 403; least-privilege Lambda role scoped to single actions and prefixes; SSE on SQS and S3.
- **Reliability:** Fail closed on Rekognition errors (unpublished beats unmoderated); Lambda DLQ with replay; idempotent conditional writes absorb duplicate EventBridge deliveries.
- **Cost Optimization:** The review percentage — not compute — is the cost driver, and the prompt forces that math; on-demand DynamoDB; Rekognition free tier (1,000 images/month, first 12 months) covers dev.
- **Operational Excellence:** Thresholds live in Parameter Store and tune without deploys; calibration script + overturn-rate loop make threshold changes evidence-based; model-version-change alarm catches silent updates.
- **Performance Efficiency:** Fully event-driven serverless; reserved concurrency sized to the Rekognition TPS quota prevents self-inflicted throttling.

## Cost Notes

Per 100,000 images, us-east-1:

| Item | Cost |
|---|---|
| Rekognition DetectModerationLabels ($0.0010/image, first 1M/mo tier) | $100.00 |
| Lambda (100k invocations, 512 MB, ~1.2s avg) | ~$1.02 |
| S3 → EventBridge events ($1.00/million) | $0.10 |
| S3 requests (PUT/COPY/tagging @ $0.005/1k) | ~$1.00 |
| DynamoDB on-demand writes ($0.625/million) | ~$0.07 |
| SQS standard | <$0.01 |
| **Automated total** | **~$102** |
| Human review @ 3.2% (3,200 images): SQS queue + internal staff | ~$267 labor (10.7h @ $25/h) |
| — or A2I private workforce | $96 fee (3,200 × $0.03) + same labor |

The honest headline: at a 3.2% routing rate, human review costs **2.6× the entire automated pipeline**. Each +1 point of review rate adds ~3.3 labor-hours (≈$83) per 100k images — threshold calibration is worth more than any Rekognition volume discount. Tiers drop to $0.0008/image past 1M images/month. Stored-video moderation is a different product at $0.10/min — keep this pipeline image-only.

## Troubleshooting

1. **InvalidImageFormatException on mobile uploads** — Cause: Rekognition accepts only JPEG and PNG; iPhones upload HEIC by default. Fix: transcode to JPEG (Pillow/Sharp) before moderation; route untranscodable files to human review — never auto-allow.
2. **ProvisionedThroughputExceededException during spikes** — Cause: DetectModerationLabels TPS quota exceeded (defaults vary by region). Fix: keep Lambda reserved concurrency below the quota, set boto3 retry mode to adaptive, request an increase in Service Quotas.
3. **Review queue balloons to 8–10% of uploads** — Cause: contextual thresholds too low for your content mix (fitness/beach-heavy platforms). Fix: rerun `calibrate.py`, raise the contextual review threshold from 75 to 85, recheck overturn rate.
4. **Flag rate doubles overnight with flat traffic** — Cause: AWS shipped a new moderation model (ModerationModelVersion changed in your verdict log) or you're being raided. Fix: recalibrate if the version changed; if not, tighten upload rate limits — you're under attack.
5. **Users appeal blocked images that look fine** — Cause: even >=95 auto-block false-positives on art and medical content. Fix: the 30-day quarantine is the appeal window; re-enqueue appeals to the priority queue; raise any category whose overturn rate exceeds 40%.
