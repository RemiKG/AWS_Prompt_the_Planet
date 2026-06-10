# S3 Storage-Class & Lifecycle Optimizer: Inventory-Driven Tiering That Avoids Retrieval-Fee Traps

Move cold S3 data to the cheapest tier that survives the first restore—a class-by-class break-even table, an Inventory-driven candidate query, and production-ready lifecycle rules that never bill you twice.

## The Problem

A 90 TB bucket in S3 Standard at $0.023/GB-month costs about $2,120/month. The common "fix"—a blanket rule pushing everything to Glacier Deep Archive—looks like a 96% saving. Then compliance requests a 2 TB re-pull: a 12-hour restore, a $0.02/GB fee, and a 180-day minimum-storage charge on objects deleted early. The "saving" out-bills doing nothing.

Three traps cause most of the damage:

1. Small-object tax: 40 million 41 KB thumbnails in Standard-IA bill at a 128 KB minimum per object—about $64/month versus $38/month in Standard. The "cheaper" class costs 70% more.
2. Retrieval-fee traps: Glacier Instant Retrieval stores at $0.004/GB-month but reads at $0.03/GB—7.5x the storage rate. Frequently-read "cold" data costs more than Standard.
3. Minimum-duration penalties: delete or re-tier within 90 days of a Glacier transition (180 for Deep Archive) and you pay the full minimum anyway.

Few teams run the class-by-class break-even math (object size x access frequency) that decides where each prefix belongs. This prompt generates it from your real S3 Inventory data and emits a production-ready lifecycle policy that rejects money-losing transitions before they ship.

## Who This Is For

- Founders and platform engineers whose S3 bill crossed $1,000/month and is climbing.
- Teams with log, backup, ML training, or media buckets that never had a lifecycle policy.
- Anyone who got burned by a Glacier retrieval bill and turned lifecycle rules off entirely.

No FinOps background required—the prompt shows its math.

## How to Use

1. Enable daily S3 Inventory (CSV or Parquet) on the target bucket with optional fields Size, LastModifiedDate, StorageClass, IntelligentTieringAccessTier. The first report lands within 48 hours.
2. Enable S3 Storage Lens (free: 14-day metrics; advanced: 15-month history with prefix-level detail) to cross-check access patterns.
3. Open Kiro CLI, Claude Code, or any MCP-enabled assistant with read access to the Inventory destination bucket.
4. Paste the System Prompt below and replace every [BRACKETED] placeholder.
5. Let the Step 0 reality check pass BEFORE anything generates. Review the break-even table, then apply the emitted JSON with `aws s3api put-bucket-lifecycle-configuration`.

Prerequisites:
- Required Access: s3:GetInventoryConfiguration, s3:PutLifecycleConfiguration, s3:GetStorageLensConfiguration, s3:GetBucketVersioning, plus read on the Inventory destination bucket. AmazonS3ReadOnlyAccess covers analysis; apply changes with a scoped inline policy.
- Recommended Background: you can read a CSV and run an Athena query. No FinOps experience required.
- Tools Required: AWS Documentation MCP server (aws_read_documentation, aws_search_documentation); AWS CLI v2; optionally Amazon Athena.

Key Parameters: "infrequent" threshold = read <1x/30 days; small-object floor = 128 KB (skip IA and Glacier Instant Retrieval below it); minimum-duration guards = 30d IA, 90d Glacier Instant/Flexible, 180d Deep Archive; read fraction f = monthly-GB-read / total-GB; region default us-east-1.

Troubleshooting: An empty or stale Inventory report is expected for the first 48 hours—S3 generates the first manifest on its own schedule. The Step 0 hard stop forces the AI to halt rather than fabricate counts.

## System Prompt

```
# S3 Storage-Class & Lifecycle Optimization Request

You are a senior AWS storage cost engineer. Produce a production-ready storage-class and
lifecycle plan for the bucket below, grounded in REAL S3 Inventory data and CURRENT
pricing. Never guess a price. Never propose a transition that loses money.

## Target
- Bucket: [BUCKET_NAME] | Region: [REGION, default us-east-1]
- Inventory report URI: [s3://INVENTORY_DEST_BUCKET/PREFIX/]
- Monthly read volume against cold data: [GB_PER_MONTH]
- Retention constraints: [e.g. "delete after 7 years", or "none"]

## Step 0 — Verify reality FIRST (hard stop on any failure)
1. Run `aws s3api list-bucket-inventory-configurations --bucket [BUCKET_NAME]`. If none
   exists, STOP: tell the user to enable daily Inventory with fields Size,
   LastModifiedDate, StorageClass, IntelligentTieringAccessTier.
2. Confirm the latest manifest.json is under 48h old; if missing or stale, STOP — never
   invent object counts.
3. Via aws_read_documentation, confirm CURRENT [REGION] storage prices, retrieval and
   transition request fees, the 128 KB minimum-billable size, and minimum durations
   (30d IA, 90d Glacier Instant/Flexible, 180d Deep Archive). Cite every price.
4. Run `aws s3api get-bucket-versioning` — versioned buckets need noncurrent rules.

## Step 1 — Profile the data from Inventory
Group objects by (prefix, size band, age band, current class). Size bands: <128 KB,
128 KB–1 MB, 1 MB–128 MB, >128 MB. Age bands: 0–30d, 30–90d, 90–180d, 180d+. Emit the
exact Athena CREATE EXTERNAL TABLE DDL plus a candidate SELECT returning, per prefix:
object_count, total_bytes, bytes_under_128kb, p50_object_size, max_last_modified.
Copy-paste SQL only.

## Step 2 — Break-even math (object size x access frequency)
Compute the cheapest viable class per group. HARD constraints:
- p50 < 128 KB: NEVER recommend Standard-IA, One Zone-IA, or Glacier Instant Retrieval —
  the 128 KB billing floor makes them cost MORE than Standard. Keep in Standard; flag
  for packing. Intelligent-Tiering never auto-tiers sub-128 KB objects either.
- Read fraction f = [GB_PER_MONTH] / total GB. Standard-IA wins only while
  (0.023 - 0.0125) > f x 0.01 per GB. Print f and the threshold per prefix.
- Glacier Instant Retrieval ($0.004 store / $0.03 retrieve — 7.5x) ONLY for data read
  less than ~once per quarter; print the multiplier on every GIR row.
- Flexible/Deep Archive ONLY when retrieval can wait hours AND retention exceeds the
  90d/180d minimum; otherwise REJECT and state the early-delete penalty.
Emit the table: prefix, size band, bytes, f, current and recommended class + cost
(storage + retrieval), monthly saving, RETRIEVAL-FEE TRAP note. Every dollar traces to
a Step 0 citation.

## Step 3 — Emit production-ready artifacts
1. lifecycle.json for `aws s3api put-bucket-lifecycle-configuration`: per-prefix
   Transitions, ObjectSizeGreaterThan: 131072 on every IA/Glacier Filter, a 7-day
   AbortIncompleteMultipartUpload rule, noncurrent-version rules when versioned.
2. An Intelligent-Tiering configuration — opt-in Archive Access (90d) and Deep Archive
   Access (180d) — stating the $0.0025 per 1,000 objects monitoring charge.
3. A dry-run plan: prefixes moved, one-time transition request fees, first-month cost
   with minimum-duration exposure, steady-state cost.
4. A rollback note: reversing a transition re-bills storage and retrieval.

## Error Handling
- Stale/missing manifest → first report takes up to 48h; halt and state the wait.
- Versioned bucket, current-only rules → noncurrent copies keep billing at Standard.
- Retention shorter than a class minimum → reject; recommend next-cheapest compliant.
- [REGION] differs from us-east-1 → re-cite regional prices; never reuse defaults.

## Acceptance evidence
Provide the `aws s3api get-bucket-lifecycle-configuration` check command, the Storage
Lens metric (StorageBytes by storage class), a 30/60/90-day savings
checklist with dollar targets, and the closing line: "No transition in this plan loses
money under the stated read estimate."

## Rules
Imperative voice, zero hedging; insufficient data = say so and stop. Show savings NET
of retrieval, transition request, and minimum-duration costs. The plan deploys in under
10 minutes once the Inventory report is current.
A transition that loses money is a bug, not a saving. Output the artifacts, no preamble.
```

## What You Get

- Athena CREATE EXTERNAL TABLE DDL plus the Inventory-driven candidate SELECT query (by prefix, size band, age band).
- A class-by-class break-even table (object size x access frequency) across Standard, Intelligent-Tiering, IA, and all three Glacier classes—retrieval-fee traps called out per row.
- A `lifecycle.json` with ObjectSizeGreaterThan: 131072 guards, a 7-day AbortIncompleteMultipartUpload rule, and noncurrent-version handling.
- An S3 Intelligent-Tiering configuration with Archive Access and Deep Archive Access tiers opt-in.
- A first-month vs. steady-state cost projection plus a verification checklist: the get-lifecycle command, the Storage Lens metric to watch, and 30/60/90-day savings milestones.

## Example Output

Break-even table excerpt (bucket app-logs-prod, us-east-1):

logs/raw/ | p50 220 KB | 3.4 TB | f=0.004/mo | current S3 Standard $80.04/mo | recommend Glacier Instant Retrieval $13.93 store + ~$0.41 retrieval = $14.34/mo | saving $65.70/mo (82%) | TRAP CHECK: GIR retrieval is $0.03/GB (7.5x storage); safe while f<0.05.

thumbnails/ | p50 41 KB | 1.6 TB across 40M objects | DO NOT transition: all objects sit under the 128 KB minimum-billable floor; Standard-IA would bill ~$64/mo vs ~$38/mo in Standard. Keep in Standard; flag for packing.

## AWS Services Used

Amazon S3 (Standard, Standard-IA, One Zone-IA, Intelligent-Tiering, Glacier Instant/Flexible/Deep Archive), S3 Inventory, S3 Storage Lens, S3 Lifecycle, Amazon Athena, AWS Cost Explorer, AWS Identity and Access Management.

## Well-Architected Alignment

- Cost Optimization: "select the most cost-effective resource type" made executable—the cheapest tier that survives the access pattern, net of retrieval fees and minimum-duration penalties.
- Operational Excellence: lifecycle rules are declarative, version-controlled JSON; savings are observable via Storage Lens, not assumed.
- Reliability: One Zone-IA only for reproducible, non-critical data—durability preserved for irreplaceable objects.
- Performance Efficiency: Intelligent-Tiering keeps latency-sensitive reads in the Frequent Access tier with no retrieval fees.
- Sustainability: cold data lands in lower-energy archive tiers, shrinking the rarely-accessed footprint.

## Cost Notes

- S3 Standard: $0.023/GB-month. Standard-IA: $0.0125 + $0.01/GB retrieval. One Zone-IA: $0.01 + $0.01/GB retrieval. Glacier Instant Retrieval: $0.004 + $0.03/GB retrieval. Glacier Flexible Retrieval: $0.0036 + $0.01/GB standard retrieval (bulk is free per-GB). Glacier Deep Archive: $0.00099 + $0.02/GB standard retrieval (us-east-1, 2026; re-verified before quoting).
- Intelligent-Tiering monitoring: $0.0025 per 1,000 objects/month—negligible above 128 KB, dominant at tens of millions of tiny objects.
- A 90 TB bucket optimized from all-Standard ($2,120/mo) to a mixed Glacier Instant/Flexible profile typically lands near $400–$600/mo—a $1,500+/mo saving without the retrieval-bill surprise, because money-losing transitions are rejected before they ship.

## Troubleshooting

- Standard-IA bill went UP after transition. Cause: objects below 128 KB are billed as 128 KB each. Fix: add ObjectSizeGreaterThan: 131072 to the lifecycle Filter (emitted by default); keep small objects in Standard or pack them.
- Surprise retrieval charge after a Glacier restore. Cause: Glacier Instant Retrieval bills $0.03/GB to read; Flexible/Deep Archive add restore fees. Fix: keep f under the printed threshold; for read-heavy "cold" data use Intelligent-Tiering (no retrieval fees).
- Charged a full minimum despite re-tiering quickly. Cause: 90-day (Glacier) / 180-day (Deep Archive) minimum-duration penalty. Fix: only transition data whose retention exceeds the minimum; the prompt rejects violations.
- Rules applied but objects didn't move. Cause: transitions run on S3's daily schedule, only on objects meeting the age/size Filter. Fix: wait 24–48h, check StorageBytes by class in Storage Lens, and verify the Filter prefix.
