# Graviton (Arm64) Migration Runbook for ECS, EKS, Lambda & RDS

Move workloads to AWS Graviton runtime-by-runtime with a compatibility scan, multi-arch images, CodeDeploy canaries, and per-service rollback — so you capture the ~20% price-performance win without a 2 a.m. "exec format error" outage.

## The Problem

AWS publishes the basis plainly: Arm64 Lambda is priced **20% lower per GB-second** than x86, Fargate Arm64 compute runs **about 20% below** x86 Fargate, and EC2 Graviton delivers **up to 40% better price performance** than comparable x86 instances (aws.amazon.com/ec2/graviton). For a startup spending $8,000/month on eligible Fargate and RDS, that is roughly **$1,600/month — $19,200/year** — left on the table. Yet the migration is full of silent traps: an x86-only base image throws `exec format error` only in production; a native dependency (`bcrypt`, `grpc`) ships no Arm64 binary; CI builds `linux/amd64` by default so the "Arm64" task silently runs emulated and slower; and RDS `db.m5` -> `db.m6g` skips the snapshot gate that protects your data. This prompt turns the migration into a boring, auditable, runtime-scoped runbook with rollback baked into every step.

## Who This Is For

- Startup platform / DevOps engineers on ECS Fargate, EKS, or Lambda who want the ~20% savings without downtime.
- RDS / Aurora (PostgreSQL, MySQL, MariaDB) teams moving to db.m6g/db.r6g/db.t4g behind a snapshot-gated cutover.
- Anyone told to "just change the CPU architecture" who needs the real checklist, build, and rollback.

## How to Use

1. Open Kiro CLI, Claude Code, or Amazon Q Developer CLI in the repo holding your Dockerfiles, task definitions, or SAM/CDK templates.
2. Paste the **System Prompt** below into a new session (or save it as `.kiro/steering/graviton-migration.md`).
3. Replace the placeholders — `[RUNTIME]` (ecs | eks | lambda | rds), `[REGION]`, `[SERVICE_NAME]`, `[CURRENT_INSTANCE_OR_ARCH]`.
4. Let the assistant run the compatibility scan FIRST and review the PASS/FAIL checklist before it writes any build files.
5. Apply the generated build, config, and rollback procedure in a non-production environment first.

**Prerequisites:**
- **Required Access:** `ecs:RegisterTaskDefinition`/`ecs:UpdateService`; `AmazonEC2ContainerRegistryPowerUser` for ECR push; `lambda:UpdateFunctionConfiguration`/`lambda:UpdateAlias`; `rds:CreateDBSnapshot`/`rds:ModifyDBInstance`; CloudWatch read.
- **Recommended Background:** Comfort with Docker, the target runtime, and `git revert`.
- **Tools Required:** Docker with Buildx v0.10+, AWS CLI v2, and the AWS Documentation MCP server (`aws_search_documentation`, `aws_read_documentation`) to verify instance families and Graviton class names against live docs.

**Key Parameters:** Buildx platforms `linux/amd64,linux/arm64` (x86 kept behind the same tag for instant rollback); ECS canary `CodeDeployDefault.ECSCanary10Percent15Minutes`, `minimumHealthyPercent=100`/`maximumPercent=200`; Lambda canary `CodeDeployDefault.LambdaCanary10Percent10Minutes`; auto-rollback alarms p99 +20% vs 7-day baseline, error rate >1% (5-min window); RDS snapshot retained 7 days, `--no-apply-immediately`.

**Troubleshooting:** If a container exits with `exec format error` on Arm64, it is expected — a `FROM` stage has no `arm64` manifest entry (common with vendor images pinned to an `amd64` digest); the scan phase catches this before you deploy.

## System Prompt

```
# Graviton (Arm64) Migration Runbook — Requirements Document

You are a senior AWS platform engineer executing a production Graviton (Arm64) migration. Migrate [SERVICE_NAME] on [RUNTIME] (ecs | eks | lambda | rds) in [REGION] from [CURRENT_INSTANCE_OR_ARCH] to Graviton with zero downtime, capturing the published Arm64 price advantage (Lambda 20% lower per GB-second; Fargate Arm64 ~20% below x86). Produce a complete, copy-paste-ready, production-ready runbook. Be imperative and specific. No hedging.

## Reality Verification (FIRST — before any build or config)
1. Via the AWS Documentation MCP server (aws_search_documentation, aws_read_documentation), confirm the Graviton target is available in [REGION] — ECS Fargate ARM64 runtimePlatform, EKS m7g/c7g node groups, Lambda arm64, or RDS db.m6g/db.r6g/db.t4g — and that the engine or runtime version supports Arm64.
2. Inspect the repo: every Dockerfile FROM stage (including pinned digests), the task definition / Helm values / SAM or CDK template, and dependency manifests.
3. Never assume a resource exists. If a region, class, or version cannot be confirmed, state the gap and ask before proceeding.

## Phase 1 — Compatibility Scan Checklist (every line PASS / FAIL / NEEDS-REVIEW)
- Every FROM image has an arm64 entry in its manifest list (docker buildx imagetools inspect); flag amd64-only digests.
- Arm64-risky native packages (bcrypt, grpc, sharp, pillow, numpy/scipy C extensions, confluent-kafka, node-gyp builds) have arm64 wheels/binaries.
- The language runtime ships Arm64 builds at the pinned version; sidecars (APM, log shippers, mesh) have arm64 images.
- RDS: engine + version supports Graviton classes; no x86-specific extensions; Multi-AZ / replica topology documented.
- For each FAIL, give the concrete fix: swap the base image, pin a multi-arch tag, add a toolchain build stage, or upgrade the dependency.

## Phase 2 — Migration (per runtime)
- ecs/eks: docker buildx create --use; docker buildx build --platform linux/amd64,linux/arm64 --tag <ECR_URI>:<tag> --push . so BOTH architectures sit behind one tag (x86 stays for instant rollback); verify with imagetools inspect. Emit a GitHub Actions snippet (setup-qemu-action + setup-buildx-action + build-push-action, platforms: linux/amd64,linux/arm64) so CI never ships amd64-only. ECS: runtimePlatform { cpuArchitecture: ARM64 }. EKS: m7g/c7g node group (amiType AL2023_ARM_64_STANDARD) or Karpenter requirement kubernetes.io/arch: arm64.
- lambda: aws lambda update-function-configuration --architectures arm64 (or SAM/CDK); rebuild every layer and native dependency for Arm64.
- rds: aws rds create-db-snapshot --db-snapshot-identifier pre-graviton-<id>-<date> first (retain 7 days); then aws rds modify-db-instance --db-instance-class <Graviton class> --no-apply-immediately so it lands in the maintenance window; require Multi-AZ.

## Phase 3 — Safe Cutover & Acceptance Evidence
- ECS: CodeDeploy blue/green with CodeDeployDefault.ECSCanary10Percent15Minutes, minimumHealthyPercent=100 / maximumPercent=200. Lambda: weighted alias via CodeDeployDefault.LambdaCanary10Percent10Minutes. EKS: rolling update maxUnavailable=0, maxSurge=25%.
- CloudWatch alarms AUTO-ROLLBACK on p99 latency +20% vs 7-day baseline OR error rate >1% over 5 minutes.
- Required evidence: uname -m via ECS Exec / kubectl exec returns aarch64; imagetools inspect shows both platforms; before/after p50/p99 latency and error rate; canary reached 100%.

## Phase 4 — Per-Service Rollback (tested, under 5 minutes)
- ECS: aws ecs update-service --task-definition <family>:<previous-revision> — amd64 is still behind the same tag, no rebuild. EKS: kubectl rollout undo deployment/<name>; cordon Graviton nodes. Lambda: aws lambda update-alias --function-version <previous>. RDS: modify back to x86 or restore the snapshot; state RPO/RTO.

## Error Handling (Cause -> Resolution)
- exec format error -> amd64-only image on an Arm64 host -> fix the flagged FROM stage, rebuild multi-arch.
- Native module fails to compile -> no arm64 prebuilt -> upgrade the dependency or add a toolchain stage.
- Arm64 task slower than x86 -> QEMU emulating amd64; arm64 layer never pushed -> verify, re-push.
- RDS reboots unexpectedly -> class change applied immediately on single-AZ -> --no-apply-immediately, Multi-AZ, maintenance window.

## Deliverables (ALL)
1. Scan checklist. 2. Updated Dockerfile(s) + exact buildx commands. 3. CI snippet building both architectures. 4. Updated task definition / Lambda config / node group / RDS modify plan. 5. CloudWatch auto-rollback alarms. 6. Per-service rollback procedure. 7. Cost comparison: x86 vs Graviton monthly, delta in dollars.

## Output Format
Clean markdown, phases in order, commands in fenced blocks, no preamble. The bar: each service migrates in under 60 minutes with a tested rollback in under 5 minutes — production-ready, no exceptions.
```

## What You Get

- A **per-runtime compatibility scan checklist** (PASS / FAIL / NEEDS-REVIEW) covering base images, native dependencies, runtimes, sidecars, and RDS engine support.
- The **multi-arch build**: updated Dockerfile(s), exact `docker buildx` commands, and a CI snippet so pipelines never silently ship amd64-only.
- The **updated runtime config** (ECS `runtimePlatform`, EKS node group or Karpenter, Lambda `Architectures: [arm64]`, RDS modify plan) plus **named CodeDeploy canaries** with CloudWatch auto-rollback alarms.
- A **per-service rollback procedure** — tested, under 5 minutes — a dollar-quantified **cost comparison**, and **acceptance evidence** proving tasks actually run aarch64 with latency/error rates held.

## Example Output

```
## Phase 1 — Compatibility Scan: payment-api (ECS Fargate, us-east-1)
[PASS] Base image node:20-slim -> arm64 manifest present
[FAIL] 'grpc@1.58' -> no prebuilt arm64 binary; fix: upgrade to grpc@1.62 or add build stage

## Phase 2 — Multi-arch build
docker buildx build --platform linux/amd64,linux/arm64 \
  --tag 1234.dkr.ecr.us-east-1.amazonaws.com/payment-api:1.9.0 --push .

## Acceptance
uname -m (ECS Exec) -> aarch64; canary 100%; p99 268ms vs 271ms baseline
x86: 6 x 1vCPU/2GB Fargate ≈ $213/mo  ->  Graviton ≈ $171/mo (~20% saved)
```

## AWS Services Used

Amazon ECS (Fargate), Amazon EKS, AWS Lambda, Amazon RDS, Amazon Aurora, Amazon ECR, AWS Graviton (Arm64), Amazon EC2 (m7g/c7g/m6g/r6g/r7g), Amazon CloudWatch, AWS CodeDeploy, AWS CLI v2, Docker Buildx, AWS Documentation MCP Server.

## Well-Architected Alignment

- **Cost Optimization:** Captures the published Graviton price advantage and requires a dollar-quantified cost comparison; the x86 image survives only as a rollback artifact.
- **Operational Excellence:** A scripted, auditable runbook with a scan gate, named CodeDeploy canary configs, and a tested sub-5-minute rollback.
- **Reliability:** Snapshot-gated RDS changes (7-day retention, `--no-apply-immediately`), CloudWatch auto-rollback, and multi-arch images that make rollback instant.
- **Performance Efficiency:** Verifies the runtime actually runs aarch64 and captures before/after p50/p99 latency as acceptance evidence.
- **Sustainability:** Graviton uses up to 60% less energy than comparable EC2 instances for the same performance.

## Cost Notes

- At ~$8,000/month on eligible Fargate + RDS, ~20% is roughly **$1,600/month ($19,200/year)**.
- Fargate Arm64 vCPU in us-east-1 is priced 20% below x86 ($0.03238 vs $0.04048/hour): 6 x (1 vCPU / 2 GB) tasks drop from **$213** to **$171/month** — **~$42/month per service**.
- Lambda Arm64 duration bills **20% lower per GB-second** — a $300/month function lands near **$240/month** after the switch and an Arm64 rebuild.

## Troubleshooting

- **`exec format error` at container start.** A `FROM` stage lacks an arm64 manifest or CI built `linux/amd64` only. Fix in the scan phase; build with `--platform linux/amd64,linux/arm64`.
- **Native module fails to compile.** No prebuilt Arm64 wheel for a pinned dependency (grpc, bcrypt). Upgrade to a version shipping arm64 artifacts or add a toolchain build stage.
- **Task runs but is slower than x86.** QEMU is emulating amd64 — the arm64 layer was never pushed. Verify `uname -m` returns `aarch64`; re-push.
- **RDS modify triggers an unexpected reboot.** The class change applied immediately or hit a single-AZ instance. Use Multi-AZ with `--no-apply-immediately`; snapshot first.
- **Lambda still bills at x86 rate.** A version/alias points to the old build or a layer was not rebuilt. Confirm `Architectures: [arm64]` and repoint the alias.
