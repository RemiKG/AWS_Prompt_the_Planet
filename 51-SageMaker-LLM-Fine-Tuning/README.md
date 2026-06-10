# Fine-Tune Open-Weight LLMs on SageMaker Managed Spot: Interruption-Proof S3 Checkpoints and a Pre-Deploy Eval Gate

Spot-priced SageMaker training jobs that checkpoint to S3 every 10 GPU-minutes, resume automatically after reclaims, and gate Model Registry approval on a hard evaluation bar—so your fine-tuning budget buys shipped models, not lost steps.

## The Problem

Fine-tuning a 7B open-weight model is now a one-day job: roughly 8 GPU-hours for a QLoRA pass over 10,000 examples. But teams running their first SageMaker training jobs lose money in three predictable ways. First, they pay On-Demand because Spot feels risky—$12.12 instead of ~$4.50 for that 7B run, and $370+ instead of ~$130 for a 13B full pass on ml.p4d.24xlarge. Second, when they do try Spot, the training script writes checkpoints to `/tmp` or `/opt/ml/model`, SageMaker never syncs them, and one capacity reclaim restarts the run from global step 0—a single botched 13B attempt burns $100+ of dead GPU-hours. Third, the run "finishes," the loss curve looks fine, and the team deploys a model that scores 4 points worse on general reasoning than the base model because nobody ran an eval before handing it to inference. This prompt produces a production-ready pipeline that fixes all three: AWS-documented managed Spot savings of up to 90% (budget 60-70% for GPU pools), checkpoint cadence sized against the 2-minute Spot reclaim warning, and an eval harness with a pass bar that blocks Model Registry approval. Inference and serving are deliberately out of scope—this pipeline ends at an approved model package your serving stack can pick up.

## Who This Is For

ML platform engineers and founding ML hires at seed-to-Series-A startups standing up their first fine-tuning capability on SageMaker; teams customizing Llama 3.1 8B, Mistral 7B, or 13B-class models on proprietary data who own training (a separate team or stack owns inference); anyone who has lost a Spot training run to a missing checkpoint and refuses to let it happen twice.

## How to Use

1. Save the System Prompt below as `finetune-request.md` and replace the bracketed placeholders: `[MODEL_ID]`, `[REGION]`, `[BUCKET]`, `[JOB_NAME]`, `[EVAL_DATASET]`.
2. Open Kiro CLI or Claude Code in an empty project directory with the AWS Documentation MCP server configured, then paste the prompt (or reference the file).
3. Let the assistant complete its preflight—caller identity, Spot quota, bucket, DLC version, current pricing—before accepting any generated code. If it skips the quota check, tell it to execute "Verify Reality" step 2 explicitly.
4. Review `cost_estimate.md`, then run the 30-minute smoke run plus the interruption drill from `runbook.md` (stop the job mid-run, relaunch, confirm resume from a nonzero step).
5. Launch the full run. When it completes, read `eval_report.json`—only a model package with `ModelApprovalStatus=Approved` proceeds to your inference pipeline.

**Prerequisites**

- **Required Access:** AWS account with SageMaker, S3, CloudWatch, and Service Quotas permissions (AmazonSageMakerFullAccess is acceptable in a dev account; scope down for production); an S3 bucket in the training region; SageMaker Spot training quota >= 1 for your instance type; a Hugging Face token in AWS Secrets Manager if the base model is gated.
- **Recommended Background:** Python, Hugging Face Trainer/PEFT basics, familiarity with launching SageMaker training jobs.
- **Tools Required:** Kiro CLI or Claude Code; AWS Documentation MCP server (aws_search_documentation, aws_read_documentation); AWS CLI v2; Python 3.10+ with sagemaker >= 2.200.

**Key Parameters:** model_id (meta-llama/Llama-3.1-8B-Instruct), instance type (ml.g5.2xlarge 7B QLoRA / ml.g5.12xlarge 13B QLoRA / ml.p4d.24xlarge full fine-tune), use_spot_instances (True), max_run/max_wait (86400s/129600s), save_steps (<=10 GPU-minutes per interval), save_total_limit (2), eval pass bar (+3.0 task points, <=1.0 harness regression), per-run budget ($15 for 7B QLoRA including eval).

**Troubleshooting:** If the first launch fails instantly with ResourceLimitExceeded, that is expected on fresh accounts—the Spot training quota ("ml.g5.2xlarge for spot training job usage") is tracked separately from the On-Demand training quota and frequently defaults to 0. Request the increase in Service Quotas, wait for approval, and relaunch.

## System Prompt

```
# Open-Weight LLM Fine-Tuning Pipeline on SageMaker — Architecture & Code Request

## Project Overview
Build a production-ready fine-tuning pipeline for an open-weight LLM (default [MODEL_ID]: meta-llama/Llama-3.1-8B-Instruct; must also support Mistral-7B and 13B-class models) using Amazon SageMaker training jobs with managed Spot instances, checkpoints synced to Amazon S3, and an automated evaluation gate that blocks regressed models from Model Registry approval. Inference and deployment are out of scope—this pipeline ends at an approved model package.

## Verify Reality Before Generating
1. Run aws sts get-caller-identity and confirm [REGION] offers the chosen instance type for SageMaker training.
2. Check Service Quotas for the Spot-specific training quota (e.g., "ml.g5.2xlarge for spot training job usage")—it is separate from the On-Demand quota and often defaults to 0 on new accounts. If it is 0, output the exact aws service-quotas request-service-quota-increase command and STOP.
3. Confirm s3://[BUCKET] exists in [REGION] with default encryption and Block Public Access enabled. Never create resources silently.
4. Pin and print exact versions: sagemaker SDK >= 2.200, the Hugging Face PyTorch Training DLC image URI for [REGION], and transformers/peft/bitsandbytes versions. Verify current instance pricing before writing the cost estimate.

## Detailed Requirements

### 1. Managed Spot Training
- Estimator config: use_spot_instances=True, max_run=86400, max_wait=129600 (max_wait MUST be >= max_run), checkpoint_s3_uri=s3://[BUCKET]/checkpoints/[JOB_NAME]/, checkpoint_local_path=/opt/ml/checkpoints.
- AWS documents managed Spot savings of up to 90% versus On-Demand; budget on 60-70% for GPU capacity pools and report the realized figure post-run as (1 - BillableTimeInSeconds / TrainingTimeInSeconds) x 100.

### 2. Checkpoint Cadence vs. Interruption
- save_strategy="steps", with save_steps computed from measured step time so one interval costs <= 10 minutes of GPU time. EC2 Spot reclaims give a 2-minute warning—assume any in-flight checkpoint write is lost, so checkpoint write + S3 sync must finish well inside the interval.
- save_total_limit=2 to cap S3 sync volume. On container start, scan /opt/ml/checkpoints for the latest checkpoint and pass resume_from_checkpoint. A restarted job MUST NOT begin at global step 0; log "Resumed from global step N" as proof.

### 3. Instance Sizing (use this table; justify any deviation)
- 7B QLoRA 4-bit (batch 4, grad-accum 4): ml.g5.2xlarge, 1x A10G 24 GB — $1.515/hr On-Demand, us-east-1
- 7B LoRA bf16 or 13B QLoRA: ml.g5.12xlarge, 4x A10G — ~$7.09/hr
- 13B LoRA bf16 or any full fine-tune: ml.p4d.24xlarge, 8x A100 40 GB — historically ~$37.69/hr On-Demand, but AWS cut P4/P5 prices by up to 45% in June 2025, so you MUST fetch the live rate before budgeting
- Always enable gradient_checkpointing and bf16 compute; default to QLoRA unless I say otherwise.

### 4. Evaluation Gate — Hard Stop Before Deploy
- After training, run a SageMaker Processing job executing EleutherAI lm-evaluation-harness (hellaswag and arc_easy, 0-shot) plus my held-out task set [EVAL_DATASET] (minimum 200 examples) against BOTH the base and fine-tuned model.
- PASS BAR: task metric >= base model + 3.0 points AND no harness task regresses by more than 1.0 point. On pass, register the model package in SageMaker Model Registry with ModelApprovalStatus=Approved. On fail, register as Rejected, attach eval_report.json, and print the 10 worst regressions. Never approve a model that has not passed the bar.

### 5. Cost & Security Controls
- Budget: <= $15 per 7B QLoRA run including eval; emit cost_estimate.md BEFORE launch.
- SageMaker execution role scoped to s3://[BUCKET]/* only; tag every job with Project, Owner, and CostCenter; CloudWatch log retention 30 days.

## Deliverables
1. launch_finetune.py — Estimator with Spot, checkpoint, and tagging config, plus a --smoke flag (500 examples, 30 minutes)
2. train.py — Hugging Face Trainer + PEFT training script with checkpoint-resume logic
3. evaluate.py and eval_gate.py — harness execution and pass-bar enforcement
4. register_model.py — Model Registry registration carrying the approval status
5. requirements.txt with pinned versions
6. cost_estimate.md — per-run cost at On-Demand list vs. expected Spot
7. runbook.md — operations guide including an interruption drill: start the smoke run, call aws sagemaker stop-training-job at minute 15, relaunch with the same checkpoint_s3_uri, and verify resume from a nonzero global step

Output runnable code with no placeholders except those listed above, without any preamble. Acceptance: the smoke run plus interruption drill completes in under 45 minutes for under $2 and proves checkpoint resume before any full run launches.
```

## What You Get

1. `launch_finetune.py` — SageMaker Python SDK launcher with `use_spot_instances=True`, `max_run`/`max_wait`, `checkpoint_s3_uri`, tags, and a `--smoke` mode.
2. `train.py` — Hugging Face Trainer + PEFT script that writes to `/opt/ml/checkpoints`, keeps `save_total_limit=2`, and resumes from the latest checkpoint on restart.
3. `evaluate.py` — runs lm-evaluation-harness plus your held-out task set against base and fine-tuned models in a SageMaker Processing job.
4. `eval_gate.py` — enforces the pass bar and emits `eval_report.json`.
5. `register_model.py` — registers the model package with `Approved` or `Rejected` status in SageMaker Model Registry.
6. `requirements.txt` — pinned sagemaker/transformers/peft/bitsandbytes versions.
7. `cost_estimate.md` and `runbook.md` — pre-launch cost projection and the interruption drill that proves resume works.

## Example Output

```
Preflight: account 123456789012, region us-east-1
  Quota "ml.g5.2xlarge for spot training job usage": 2 ........ OK
  Bucket s3://acme-ml-train (us-east-1, SSE, BPA on) ......... OK

Launch: llama31-8b-qlora-20260610 (Spot, max_run 86400 / max_wait 129600)
  checkpoint_s3_uri: s3://acme-ml-train/checkpoints/llama31-8b-qlora-20260610/
  save_steps: 120 (~9.4 GPU-min per interval), save_total_limit: 2

Run summary: TrainingTimeInSeconds 31,200 | BillableTimeInSeconds 10,920
  Realized Spot savings: 65% | Cost: $4.59 vs $13.13 On-Demand
  1 interruption at step 1,440; resumed from global step 1,440 (0 steps lost)

Eval gate: task exact-match 71.4 vs base 64.1 (+7.3 PASS)
  hellaswag -0.4 PASS | arc_easy +0.2 PASS
  Model package llama-ft/3 -> ModelApprovalStatus=Approved
```

## AWS Services Used

Amazon SageMaker (Training Jobs with managed Spot, Processing Jobs, Model Registry), Amazon S3, Amazon CloudWatch, AWS Identity and Access Management (IAM), AWS Service Quotas, AWS Secrets Manager.

## Well-Architected Alignment

- **Cost Optimization:** managed Spot training (AWS-documented up to 90% savings; 60-70% budgeted for GPU pools), `save_total_limit=2` to cap checkpoint storage, a $15-per-run budget enforced via `cost_estimate.md` before launch, and realized savings reported from `BillableTimeInSeconds`.
- **Reliability:** checkpoint cadence sized against the 2-minute Spot reclaim warning, deterministic resume from S3, and a mandatory interruption drill that proves recovery before real money is spent.
- **Operational Excellence:** an eval harness with a hard pass bar gating Model Registry approval, `eval_report.json` as a release artifact, a runbook, and 30-day CloudWatch log retention.
- **Security:** execution role scoped to a single bucket prefix, S3 default encryption with Block Public Access, gated-model tokens in Secrets Manager, and mandatory Project/Owner/CostCenter tags.
- **Performance Efficiency:** right-sized instances per model size (7B/13B sizing table), QLoRA 4-bit by default, gradient checkpointing and bf16 everywhere.

## Cost Notes

- 7B QLoRA (10k examples, 3 epochs, ~8 GPU-hours) on ml.g5.2xlarge: $12.12 On-Demand → roughly $3.60-$4.85 at typical 60-70% Spot savings.
- 13B QLoRA (~10 hours) on ml.g5.12xlarge: $70.90 On-Demand → roughly $21-$28 on Spot.
- Full fine-tunes on ml.p4d.24xlarge historically listed at ~$37.69/hr On-Demand in us-east-1; AWS cut P4/P5 prices by up to 45% in June 2025, so confirm the current rate before budgeting.
- Eval Processing job: ~1 hour on ml.g5.2xlarge ≈ $1.52—run it On-Demand; it is short and it gates the release.
- Checkpoints: two retained checkpoints ≈ 4 GB for a 7B QLoRA run ≈ $0.09/month at S3 Standard ($0.023/GB-month).
- Basis for the discount claim: AWS's managed Spot training documentation states savings of up to 90% versus On-Demand; SageMaker reports each job's actual figure as (1 − BillableTimeInSeconds / TrainingTimeInSeconds) × 100.

## Troubleshooting

1. **ResourceLimitExceeded on CreateTrainingJob.** Cause: the Spot training quota (e.g., "ml.g5.2xlarge for spot training job usage") is separate from On-Demand and often 0 on new accounts. → Fix: request the Spot-specific quota in Service Quotas, then relaunch.
2. **Job restarts from step 0 after a Spot reclaim.** Cause: checkpoints written to `/opt/ml/model` or `/tmp` instead of `checkpoint_local_path` (`/opt/ml/checkpoints`), or `resume_from_checkpoint` never wired. → Fix: write to `/opt/ml/checkpoints`, scan it at container start, and prove resume with the interruption drill before any full run.
3. **CUDA out of memory on a 13B model.** Cause: bf16 LoRA attempted on a single A10G, or gradient checkpointing disabled. → Fix: switch to 4-bit QLoRA with `gradient_checkpointing` and batch 1 + grad-accum 16, or move up to ml.g5.12xlarge per the sizing table.
4. **Job stops after exceeding MaxWaitTimeInSeconds without finishing.** Cause: Spot capacity for the instance family is scarce and `max_wait` is too tight. → Fix: raise `max_wait` to 1.5-2x `max_run`, try another region with capacity, or fall back to On-Demand for deadline-bound runs.
5. **Eval gate rejects the model on a harness regression.** Cause: catastrophic forgetting from a too-high learning rate or too many epochs. → Fix: drop the LR to 1e-4, cut to 2 epochs, and mix 5-10% general instruction data into the training set.

Bottom line: smoke run plus interruption drill in under 45 minutes for under $2, a full 7B fine-tune for under $5 of Spot compute, and nothing reaches your inference team without an Approved eval gate.
