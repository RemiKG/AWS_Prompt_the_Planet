# Real-Time Clickstream Pipeline: Kinesis to Firehose to S3 Iceberg to Athena (No Lost Events, No Idle Spend)

Turn raw click events into SQL-queryable Apache Iceberg tables with sub-minute latency—so you ship behavioral analytics this sprint instead of patching a Kafka cluster for six weeks.

## The Problem

A 2-person analytics startup wants to capture 2,000 clickstream events/second (peaks to 5,000/s) and query them in SQL within a minute. Self-managed Kafka means 3x broker EC2 instances, ZooKeeper or KRaft, manual rebalancing, and $900-1,500/month of compute billing the same at 3 AM as at peak. Teams then under-provision shards, hit `ProvisionedThroughputExceededException`, and silently drop events.

The serverless path—Kinesis Data Streams plus Amazon Data Firehose delivering to S3 Iceberg tables, queried by Athena—carries zero idle compute and runs roughly a third of the Kafka bill. Three traps wreck production deployments: (1) shard count is a math problem (1 shard = 1 MB/s OR 1,000 records/s) and getting it wrong throttles writes; (2) Firehose buffering trades latency against cost against small files; (3) delivery is at-least-once, NOT exactly-once—claim otherwise and your dashboard double-counts. This prompt makes an AI assistant generate the whole production-ready pipeline with the shard math shown, buffering tuned, and duplicates handled honestly.

## Who This Is For

- Founding or solo data engineers shipping clickstream analytics without running streaming infrastructure.
- Backend teams that know SQL and S3 but have never sized a Kinesis stream.
- Anyone burned by Firehose writing thousands of tiny S3 files that slowed Athena scans.

## How to Use

1. Open Kiro CLI, Claude Code, or any AI assistant with the AWS Documentation MCP server connected.
2. Paste the System Prompt below verbatim.
3. Replace every bracketed placeholder `[LIKE_THIS]`: events/sec (average + peak), event size, Region, bucket name, max query latency.
4. Let the assistant VERIFY reality first, then review the shard-math and duplicate-handling sections.
5. Deploy with `terraform init && terraform apply`.

Prerequisites:
- Required Access: rights to create Kinesis/Firehose streams, Glue databases, S3 buckets, IAM roles, CloudWatch alarms (`AdministratorAccess` first, then scope down).
- Recommended Background: basic SQL, Terraform, CloudWatch fundamentals.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`), Terraform >= 1.6, AWS CLI v2.

Key Parameters: shards from `ceil(max(peak_events_per_sec/1000, peak_MBps))` (5,000/s at 1.2 KB = 6), Firehose buffer 64 MiB / 60 s (range 1-128 MiB / 0-900 s), 7-day lifecycle on Athena results, Glue auto-compaction on, KMS SSE, alarms on `WriteProvisionedThroughputExceeded` and `KinesisMillisBehindLatest`.

Troubleshooting: Firehose-to-Iceberg throughput stalling at 1 MiB/s outside us-east-1, us-west-2, and eu-west-1 is EXPECTED—the 5 MiB/s Direct PUT cap exists only in those Regions. Set `AppendOnly=True` at stream creation or file a limit-increase request.

## System Prompt

```
# Real-Time Clickstream Pipeline — Infrastructure Design Request

You are a senior AWS data engineer. Produce production-ready Terraform (>= 1.6) for a fully serverless clickstream pipeline: Amazon Kinesis Data Streams -> Amazon Data Firehose -> Apache Iceberg tables on Amazon S3 -> Amazon Athena. Optimize for zero idle compute and sub-minute query latency. Do NOT generate code until VERIFY REALITY FIRST is complete and reported.

## VERIFY REALITY FIRST (mandatory, before any code)
Using the AWS Documentation MCP tools (aws_search_documentation, aws_read_documentation), confirm and report as a "Reality Check" section:
1. Firehose supports Apache Iceberg Tables as a destination in [AWS_REGION]; if not, STOP and name the nearest supported Region.
2. The Iceberg-destination throughput cap there (5 MiB/s Direct PUT in us-east-1/us-west-2/eu-west-1, 1 MiB/s elsewhere) and the exact data-freshness metric Firehose emits for it.
3. The exact Apache Iceberg library version Firehose currently supports.
4. Whether bucket [S3_BUCKET_NAME] exists, plus the Glue database/table names you will use (hyphens are invalid — use underscores).
Never invent a version, quota, metric name, or price — if the docs do not confirm it, say so and use a documented default.

## Detailed Requirements

### 1. Shard Math (show your work)
- Inputs: [EVENTS_PER_SEC] average, [PEAK_EVENTS_PER_SEC] peak, [AVG_EVENT_SIZE_KB] per event. One shard = 1,000 records/s OR 1 MB/s, whichever is hit first.
- shards = ceil( max( PEAK_EVENTS_PER_SEC / 1000 , PEAK_EVENTS_PER_SEC * AVG_EVENT_SIZE_KB / 1024 ) ). Show both terms and the chosen max.
- Recommend Provisioned for stable load, On-Demand for spiky/unknown; state the dollar trade-off ($0.015/shard-hr + $0.014/M PUT payload units vs $0.04/stream-hr + $0.08/GB-in, us-east-1). Recommend KPL-style aggregation when events are under 5 KB.

### 2. Firehose Buffering Trade-offs (state them plainly)
- Buffer 1-128 MiB / 0-900 s; default 64 MiB / 60 s. Smaller = lower latency but MANY tiny S3 files = slow, costly Athena scans; larger = cheaper queries, higher latency.
- Pick values satisfying [MAX_QUERY_LATENCY_SEC] end-to-end; justify in one sentence.

### 3. Delivery Guarantee (be brutally honest)
- Firehose-to-Iceberg is AT-LEAST-ONCE, NOT exactly-once. Iceberg commits use Optimistic Concurrency Control; retries can produce DUPLICATES and exhausted failures land in the S3 error prefix.
- Deliver an idempotency strategy: a stable event_id stamped at the producer plus an Athena dedupe view (ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY ingest_ts)) or MERGE on event_id. Never claim exactly-once anywhere.

### 4. Cost Discipline
- Idle compute MUST be $0 — no EC2, no EMR, no always-on consumers.
- Emit a monthly cost table at [EVENTS_PER_SEC] average using current [AWS_REGION] rates; flag any line over $100/month and name the cheaper lever (aggregation, Provisioned vs On-Demand, larger buffers).
- Expire Athena query results after 7 days via S3 lifecycle; enable Glue automatic compaction.

### 5. Security & Reliability
- KMS encryption on stream and bucket; TLS in transit. Least-privilege Firehose role scoped to exact Glue, S3, KMS, and Logs ARNs — no wildcards.
- Alarms: Kinesis WriteProvisionedThroughputExceeded > 0 (1 datapoint); Firehose KinesisMillisBehindLatest sustained above [MAX_QUERY_LATENCY_SEC] x 1000.

## Error Management (document Cause -> Resolution in the README)
1. Producer throttling: shards below peak or a hot PartitionKey -> recompute shard math, spread the key, retry with backoff.
2. Throughput pinned at 1 MiB/s outside the three 5-MiB/s Regions -> AppendOnly = true at stream creation, or a limit increase.
3. S3 error prefix records: schema/JSON mismatch, hyphenated names, or multiple writers per table (OCC) -> align schema, underscores only, one stream per table.
4. Ballooning Athena scans: small-file explosion -> raise buffer toward 128 MiB, confirm compaction, schedule OPTIMIZE ... REWRITE DATA USING BIN_PACK then VACUUM.

## Deliverables (enumerate every file)
1. main.tf — Kinesis stream, Firehose stream (Iceberg destination), Glue database + table, S3 bucket with lifecycle, KMS key.
2. iam.tf — least-privilege Firehose role + policy.
3. monitoring.tf — both alarms + a 4-widget dashboard (IncomingRecords, WriteProvisionedThroughputExceeded, KinesisMillisBehindLatest, DataReadFromKinesisStream.Bytes).
4. athena/ — table DDL (or note Firehose-managed), dedupe view, 2 sample queries (top pages, sessionized funnel).
5. variables.tf, outputs.tf, README.md — shard math, buffering rationale, at-least-once caveat, Error Management table.
6. Monthly cost table + a PutRecords producer snippet (max 500 records/call) with an explicit PartitionKey strategy.

## ACCEPTANCE / EVIDENCE (prove it works)
Provide a numbered post-deploy checklist: (a) aws kinesis put-record test command, (b) the CloudWatch metrics confirming delivery, (c) the exact Athena query returning the test row, (d) proof of zero throttling. The pipeline must deploy in under 10 minutes via terraform init && terraform apply and return a queryable test event in Athena in under 5 minutes — on any failure, emit the matching Error Management entry instead of declaring success.

Output the IaC and docs with no preamble. Begin with the Reality Check section.
```

## What You Get

- main.tf: KMS-encrypted Kinesis stream (shards from the math), Firehose stream targeting an Iceberg table, Glue database + table, S3 bucket with lifecycle.
- iam.tf: least-privilege Firehose role scoped to specific Glue, S3, KMS, and Logs ARNs.
- monitoring.tf: throttling + consumer-lag alarms, 4-widget dashboard on verified metric names.
- athena/ddl.sql, dedupe_view.sql, sample_queries.sql (top pages, sessionized funnel).
- variables.tf, outputs.tf, README.md (shard math, buffering rationale, at-least-once caveat, Cause -> Resolution table), a monthly cost table, and a PutRecords producer snippet with a partition-key strategy.

## Example Output

Reality Check: Firehose supports Apache Iceberg Tables in us-east-1 (Direct PUT cap 5 MiB/s). Iceberg library 1.5.2. Glue database clickstream_db, table events (hyphens invalid).

Shard math: peak 5,000 events/s at 1.2 KB. Records term = 5; throughput term = 5.86 MB/s -> 6; chosen max = 6 shards (Provisioned). Delivery is at-least-once — dedupe via the events_deduped view (ROW_NUMBER over event_id by ingest_ts).

## AWS Services Used

Amazon Kinesis Data Streams, Amazon Data Firehose, Amazon S3 (Apache Iceberg tables), AWS Glue Data Catalog, Amazon Athena, AWS KMS, Amazon CloudWatch, AWS IAM.

## Well-Architected Alignment

- Operational Excellence: alarms + dashboard on throttling and consumer lag; acceptance checklist; repeatable IaC.
- Security: KMS at rest, TLS in transit, least-privilege role, no wildcards.
- Reliability: shard sizing from peak with a throttling alarm; honest at-least-once with a dedupe pattern; OCC retries documented.
- Performance Efficiency: buffering tuned for sub-minute latency; compaction keeps Parquet files large and Athena fast.
- Cost Optimization: zero idle compute, Provisioned-vs-On-Demand computed, aggregation for small events, 7-day result lifecycle.
- Sustainability: no always-on brokers; compute scales with event volume.

## Cost Notes

At a sustained 2,000 events/s (1.2 KB, ~6.3 TB/month, us-east-1):
- Kinesis Provisioned: 6 shards x $0.015/shard-hour x 730h = ~$66, plus ~$73 of PUT payload units ($0.014/million). KPL-style aggregation packs ~20 small events per 25 KB unit and cuts that line 10x; Firehose de-aggregates automatically. On-Demand ($0.04/stream-hour + $0.08/GB-in) wins only for spiky traffic.
- Firehose to Iceberg: ~$0.045/GB delivered from Kinesis = ~$280/month. Athena: $5.00/TB scanned — compaction + partitioning keep it near zero. S3 + KMS: a few dollars; Glue compaction bills per DPU-hour.
- All-in: ~$400-450/month — about a third of self-managed Kafka, with zero idle compute and zero broker ops. At hobby volume, On-Demand drops to tens of dollars. Confirm live rates on the AWS pricing pages.

## Troubleshooting

- WriteProvisionedThroughputExceeded > 0. Cause: shards below peak or a hot PartitionKey. Fix: recompute shard math, add shards or go On-Demand, spread the key.
- Slow, expensive Athena queries. Cause: small buffers made thousands of tiny Parquet files. Fix: buffer 64-128 MiB, Glue compaction, OPTIMIZE then VACUUM.
- Duplicate event rows. Cause: at-least-once retries after OCC conflicts. Fix: query events_deduped or MERGE on event_id — never assume exactly-once.
- Throughput capped at 1 MiB/s. Cause: Region outside us-east-1/us-west-2/eu-west-1. Fix: AppendOnly=True at creation or a limit increase.
- Delivery to the S3 error prefix. Cause: schema/JSON mismatch, hyphenated names, or multiple streams per table. Fix: align schema, underscores, one stream per table.
