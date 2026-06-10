# SQS vs SNS vs EventBridge vs Kinesis: The Async Backbone Decision Kit

Score your workload's fan-out, ordering, replay, and throughput needs against all four AWS async primitives with verified cost-crossover math—so the backbone you ship in week one is still the right one at 10x traffic.

## The Problem

Every async architecture starts with a 30-second decision that takes six weeks to reverse. Teams default to whatever they used last, and the bill for guessing wrong is concrete:

- **EventBridge as a work queue**: at 1,000 msg/sec (2.59B events/month), EventBridge custom events cost **$2,592/month** at $1.00 per million. Batched SQS Standard (10 msgs/request) handles the same volume for **~$104/month**, and two provisioned Kinesis shards do it for **~$58/month**. That is a 25–45x premium for content routing a single-consumer queue never uses.
- **SQS Standard where ordering matters**: best-effort ordering means out-of-order delivery shows up only under production load—usually as a corrupted state machine three weeks after launch.
- **EventBridge where replay matters**: archives only capture events created *after* the archive exists. Discover this post-incident and the history you needed is permanently gone. (Kinesis retains 24 hours by default, extendable to 365 days.)
- **Kinesis for 5 msg/sec**: one provisioned shard costs $10.95/month at zero traffic and forces every consumer to manage checkpointing—2–4 engineer-weeks of code a $3/month SQS queue never needs.

Migrating between primitives after launch means rewriting consumers (poll/delete vs. shard checkpointing), re-deriving idempotency, and accepting a replay gap. This kit makes the decision *before* any of that code exists.

## Who This Is For

- Startup backend teams designing their first event pipeline who need the decision defensible in an architecture review
- Platform engineers standardizing async patterns across services and tired of four teams picking four primitives for the same shape of problem
- Teams already feeling wrong-primitive pain who need honest migration-cost math before committing to a rewrite

## How to Use

1. Save the System Prompt below as `async-decision-kit.md` in your project (Kiro: drop it in `.kiro/steering/`; Claude Code: reference it in `CLAUDE.md` or paste directly into chat).
2. Replace every bracketed placeholder `[LIKE_THIS]` with measured numbers. If you have an existing system, pull sustained/peak rates from CloudWatch (`NumberOfMessagesSent`, `IncomingRecords`, or your producer's request metrics)—never guess throughput.
3. Run the prompt. The assistant walks the four-gate decision tree, verifies current pricing and Region quotas via the AWS Documentation MCP server, then emits five artifacts.
4. Review `decision-summary.md` and `cost-model.md` first. Then `terraform init && terraform plan && terraform apply`, and run `verify.sh` to prove the backbone works end to end.

**Prerequisites**

- **Required Access**: AWS account with permissions to create the winning primitive (`sqs:CreateQueue`, `sns:CreateTopic`, `events:PutRule`, `kinesis:CreateStream`), plus CloudWatch alarms, KMS key creation, and IAM policy authoring.
- **Recommended Background**: Basic Terraform (>= 1.5), producer/consumer messaging concepts, ability to read a CloudWatch metric.
- **Tools Required**: Kiro CLI or Claude Code with the AWS Documentation MCP server enabled (`aws_read_documentation`, `aws_search_documentation`) for live pricing/quota verification; AWS CLI v2; Terraform >= 1.5.

**Key Parameters:** Sustained/peak throughput (50/400 msg/sec), payload size (1 KB), consumer count (3), ordering scope (none/per-entity/global), replay window (0–365 days), duplicate tolerance (yes/no), latency budget (1,000 ms), monthly ceiling ($150), Region (us-east-1).

**Troubleshooting:** If the assistant recommends EventBridge for a single-consumer work queue, it skipped the cost gate—re-run with your real msg/sec filled in. At 100 msg/sec, EventBridge costs ~$259/month for content routing that workload never uses; this is expected behavior when the throughput placeholder is left bracketed.

## System Prompt

```markdown
# Async Messaging Backbone Decision Request

## Project Overview
I am choosing the async messaging primitive for [WORKLOAD_NAME] BEFORE writing any
code. Candidates: Amazon SQS (Standard and FIFO), Amazon SNS, Amazon EventBridge,
and Amazon Kinesis Data Streams. Region: [REGION, default us-east-1].
Terraform preferred. Do not default to the trendiest service—score my workload
against each primitive and show your math.

## Workload Profile (score against these numbers, not vibes)
- Throughput: [MSG_PER_SEC, e.g. 50] msg/sec sustained, [PEAK_MSG_PER_SEC, e.g. 400]
  peak, [PAYLOAD_KB, e.g. 1] KB average payload
- Fan-out: [CONSUMER_COUNT, e.g. 3] independent consumers; content-based filtering
  needed: [YES/NO]
- Ordering: [NONE / PER_ENTITY / GLOBAL], keyed by [ORDER_KEY, e.g. order_id]
- Replay: reprocess [REPLAY_DAYS, e.g. 0] days of history; new consumers must read
  old data: [YES/NO]
- Delivery tolerance: duplicates acceptable: [YES/NO]; max end-to-end latency
  [LATENCY_MS, e.g. 1000] ms
- Budget ceiling for the messaging layer: [BUDGET, e.g. $150]/month

## Detailed Requirements

### 1. Decision Tree
Walk a four-gate tree in this exact order—Replay, Throughput, Fan-out, Ordering—and
record which gate eliminated each service. Gates: (a) replay window > 0 or multiple
readers of the same ordered history -> stream storage (Kinesis); (b) sustained
ingest > 1 MB/s or > 1,000 records/sec per ordered key -> shard-based scaling;
(c) more than 2 heterogeneous subscribers or content filtering -> SNS or EventBridge
fan-out into per-consumer SQS queues, never direct push to workers; (d) per-entity
ordering under 300 msg/sec per group -> FIFO variants.

### 2. Cost Crossover Math
Build a monthly cost table at exactly my stated msg/sec and payload for all four
services, plus 0.1x and 10x my volume. Verify current rates with
aws_read_documentation or the public pricing pages BEFORE quoting them; flag any
number you could not verify. State the crossover point in msg/sec where one
provisioned Kinesis shard undercuts batched SQS, state EventBridge's $1.00
per-million premium explicitly, and include the idle cost of each option at
0 msg/sec.

### 3. Honest Guarantees
Produce a per-service table of ordering, delivery semantics, deduplication, replay,
max payload, and throughput ceilings—stated as they actually are: SQS Standard
reorders under load; EventBridge never guarantees order, even on replay; Kinesis
orders only per partition key within a shard; "exactly-once" exists only as SQS
FIFO's 5-minute deduplication window; Kinesis consumers see duplicates from
producer retries and must be idempotent. No marketing language.

### 4. Migration Cost of Choosing Wrong
For my top two candidates, estimate engineer-weeks and data-loss exposure to
migrate between them later, in both directions: producer API swap, consumer
rewrite (poll/delete vs. checkpoint), and the replay gap. Recommend the topology
that minimizes regret—e.g., a publish abstraction at the producer edge and fan-out
into SQS so a later backbone swap touches one module, not every consumer.

### 5. Reality Check Before Generating
Before recommending anything: confirm the winning service and any FIFO or
high-throughput mode is available in [REGION]; check current default quotas
(PutEvents TPS, FIFO TPS, shards per account) against my stated peak; if peak
exceeds a default quota, name the exact Service Quotas increase to file before
launch. If my profile is self-contradictory (e.g., GLOBAL ordering at 10,000
msg/sec), stop and tell me instead of generating a broken design.

## Deliverables Requested
1. decision-summary.md — gate-by-gate tree walk, final recommendation, confidence
   level, and the single fact that would change the answer
2. cost-model.md — the crossover table with every formula shown
3. guarantees-matrix.md — the honest per-service limits table
4. main.tf — production-ready Terraform for the winning primitive: SSE with
   aws_kms_key, dead-letter queue or archive wired in, CloudWatch alarms
   (ApproximateAgeOfOldestMessage > 300s, GetRecords.IteratorAgeMilliseconds
   > 60000, or FailedInvocations > 0 as applicable), least-privilege IAM
   producer and consumer policies
5. verify.sh — AWS CLI commands proving it works: send 10 test messages, confirm
   receipt and sequence, print alarm states

Align the design with the Well-Architected Reliability and Cost Optimization
pillars. Output the deliverables directly without any preamble. The full kit must
be reviewable in under 15 minutes and deployable in under 10.
```

## What You Get

1. **`decision-summary.md`** — the four-gate tree walk with the eliminating gate named per service, a final recommendation with confidence level, and the single workload change that would flip the answer
2. **`cost-model.md`** — monthly cost at your stated msg/sec plus 0.1x/10x, formulas shown, crossover points and idle costs called out
3. **`guarantees-matrix.md`** — ordering, delivery, dedup, replay, payload, and throughput ceilings per service, stated honestly
4. **`main.tf`** — production-ready Terraform for the winner: KMS encryption, DLQ/archive, CloudWatch alarms, least-privilege producer/consumer IAM policies
5. **`verify.sh`** — acceptance evidence: 10 test messages sent, receipt and ordering confirmed, alarm states printed

## Example Output

```
GATE 1 — REPLAY: 0 days required. Kinesis's decisive advantage (native replay,
24h default retention, 365d max) is void. ELIMINATED: none yet, but Kinesis loses
its trump card.
GATE 2 — THROUGHPUT: 50 msg/sec @ 1 KB = 0.05 MB/s. Two orders of magnitude under
one shard's 1 MB/s ceiling. Shard-based scaling buys nothing.
GATE 3 — FAN-OUT: 1 consumer today, no content filtering. ELIMINATED: EventBridge
($1.00/M ingest = $129.60/mo here vs $6.48 for FIFO SQS — a 20x premium for
routing you don't use) and SNS (nothing to fan out to).
GATE 4 — ORDERING: PER_ENTITY on order_id, well under 300 msg/sec per group.
RECOMMENDATION: SQS FIFO, MessageGroupId = order_id. Confidence: HIGH.
Cost at your volume: $6.48/mo (12.96M batched requests x $0.50/M).
Crossover note: one Kinesis shard ($10.95/mo idle) undercuts batched SQS FIFO
above ~120 msg/sec sustained — revisit at 10x growth, where replay needs appear too.
Regret hedge: keep publishes behind one module; a later swap to Kinesis is a
2–3 week producer-edge change, not a consumer rewrite.
```

## AWS Services Used

Amazon SQS, Amazon SNS, Amazon EventBridge, Amazon Kinesis Data Streams, Amazon CloudWatch, AWS Key Management Service, AWS Identity and Access Management, Service Quotas

## Well-Architected Alignment

- **Reliability**: every design ships with a dead-letter queue or archive, alarms on `ApproximateAgeOfOldestMessage` (> 300s) or `GetRecords.IteratorAgeMilliseconds` (> 60,000 ms), and delivery semantics stated honestly so consumers are built idempotent from day one.
- **Cost Optimization**: the crossover table is a first-class deliverable; idle cost at 0 msg/sec is computed per option; batching (10 messages/request) is assumed and shown in the math.
- **Performance Efficiency**: throughput ceilings are checked against Region quotas before recommendation, and quota increases are named before launch, not after throttling.
- **Operational Excellence**: `verify.sh` produces acceptance evidence; `decision-summary.md` doubles as the architecture decision record for future reviewers.
- **Security**: SSE-KMS on every primitive, separate least-privilege IAM policies for producer and consumer roles.

## Cost Notes

Verified against public AWS pricing (us-east-1, June 2026):

- **SQS**: $0.40 per million requests (Standard), $0.50/M (FIFO); first 1M/month free; each 64 KB chunk bills as one request; max message now 1 MiB.
- **SNS**: $0.50 per million publishes; delivery to SQS and Lambda is free.
- **EventBridge**: $1.00 per million custom events (64 KB chunks); archive processing $0.10/GB, archive storage $0.023/GB-month; replays bill again at custom-event rates. AWS service events on the default bus are free.
- **Kinesis Data Streams**: provisioned $0.015/shard-hour ($10.95/month) + $0.014 per million 25 KB PUT payload units; on-demand bills per stream-hour plus per-GB data-in/out (no shard management); confirm current on-demand rates on the pricing page before quoting. Enhanced fan-out adds a per-consumer-shard-hour charge plus per-GB retrieved.

Worked example at 50 msg/sec, 1 KB (129.6M msgs/month): SQS Standard batched **$5.18** (12.96M requests x $0.40/M), SQS unbatched **$51.84**, SQS FIFO batched **$6.48** ($0.50/M), EventBridge **$129.60**, one provisioned Kinesis shard **$12.76** ($10.95 shard + $1.81 PUT). Crossover: one provisioned shard undercuts batched SQS Standard above **~160 msg/sec sustained** (batched FIFO above ~120 msg/sec, unbatched SQS above ~11 msg/sec)—but at 0 msg/sec the queue costs $0 while the shard still costs $10.95/month. Spiky, low-volume traffic favors request-billed services; sustained volume favors shards.

## Troubleshooting

1. **Assistant recommends Kinesis for a 5 msg/sec workload** — Cause: budget ceiling left bracketed, so idle shard cost was never penalized. Fix: set `[BUDGET]` and re-run; the prompt forces an idle-cost row at 0 msg/sec.
2. **`verify.sh` FIFO sends fail with `MissingParameter`** — Cause: every send to a FIFO queue requires `--message-group-id`. Fix: the generated script includes it; add it to any hand-written test sends too.
3. **EventBridge replay returns zero events** — Cause: archives only capture events created after the archive exists; there is no backfill. Fix: if replay scored > 0 in your profile, the kit creates the archive in `main.tf` at bus creation—deploy it before producers go live.
4. **Cost table disagrees with your actual bill** — Cause: payload chunking. SQS and EventBridge bill per 64 KB chunk, so a 200 KB payload is 4 billed units per message. Fix: re-run with your real `[PAYLOAD_KB]`; the formulas in `cost-model.md` make the chunk multiplier explicit.
5. **`terraform apply` fails with `AccessDenied` on KMS** — Cause: the deploying role lacks `kms:CreateKey`/`kms:CreateAlias`. Fix: grant them, or pass an existing CMK ARN via the `kms_key_arn` variable the kit exposes.
