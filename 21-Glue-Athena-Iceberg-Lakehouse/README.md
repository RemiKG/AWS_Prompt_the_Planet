# Serverless Lakehouse Starter: Glue + Athena + Iceberg with Per-Query Byte Guardrails

Stand up a production-ready serverless lakehouse on S3 with AWS Glue Data Catalog, Apache Iceberg tables, and Amazon Athena—partitioned, compacted, and capped so one careless `SELECT *` can never scan 4 TB and bill you $20.

## The Problem

Athena charges $5.00 per TB scanned (us-east-1, on-demand), so one `SELECT *` against an unpartitioned 4 TB table costs $20—and a dashboard refreshing it every 15 minutes burns ~$1,920 a day. Streaming ingestion lands millions of 4-128 KB files, and latency balloons from 2s to 90s as Athena opens files instead of reading data. Schema drift—a producer adds one column—silently breaks crawler-managed Hive tables with NULLs or `HIVE_PARTITION_SCHEMA_MISMATCH`. And with no workgroup limit, nothing stops a runaway query. This prompt generates a lakehouse with all four problems solved on day one: date-based partitioning, scheduled Iceberg compaction toward 128 MB files, safe in-place schema evolution, and an Athena workgroup that auto-cancels any query past 50 GB scanned.

## Who This Is For

- Founding/solo data engineers who need SQL analytics on S3 without Spark clusters or a warehouse.
- Backend teams landing event/log/clickstream data in S3 who don't want a $2k/month Redshift bill.
- Anyone who got bill-shock from an unbounded Athena query and wants hard guardrails before the next one.

## How to Use

1. Open Kiro CLI, Claude Code, or any AI assistant with the AWS Documentation MCP server enabled.
2. Paste the full System Prompt below into a new session.
3. Replace the placeholders: `[AWS_REGION]`, `[DATA_S3_BUCKET]`, `[DATABASE_NAME]`, `[PARTITION_COLUMNS]` (e.g. event_date, region), `[MONTHLY_QUERY_BUDGET]` (e.g. $150).
4. Describe your source data: format, daily volume, and the columns you filter on most.
5. Review the generated IaC and Athena DDL/DML, then deploy with `terraform apply` (or `aws cloudformation deploy`).
6. Run the CTAS migration, enable the scheduled OPTIMIZE/VACUUM jobs, then run the Acceptance Checklist to prove every guardrail exists.

Prerequisites:
- Required Access: an IAM principal with `glue:*` on the database, `athena:*` in the workgroup, `s3:GetObject`/`s3:PutObject` on the data bucket, `lakeformation:GrantPermissions`, and `iam:CreateRole` for the Glue service role.
- Recommended Background: basic SQL and comfort with Terraform or CloudFormation.
- Tools Required: Kiro CLI or Claude Code; AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`); AWS CLI v2; Terraform >= 1.6.

Key Parameters: Athena engine v3, Iceberg format-version 2, `BytesScannedCutoffPerQuery=53687091200` (50 GB) with `EnforceWorkGroupConfiguration=true`, SNS data-usage alert at 80% of `[MONTHLY_QUERY_BUDGET]`/30, `write_target_data_file_size_bytes=134217728` (128 MB), `vacuum_max_snapshot_age_seconds=259200` (3 days), crawler `cron(0 1 * * ? *)`, Glacier Instant Retrieval lifecycle at 90 days.

Troubleshooting: If Athena cancels a query with `Bytes scanned limit was exceeded`, the guardrail worked—a scan ignored the partition filter. Add a `WHERE` predicate on a partition column, or run intentional large scans in a separate admin workgroup.

## System Prompt

```
# Serverless Lakehouse Build Request — Glue + Athena + Iceberg

## Project Overview
Act as a senior AWS data engineer. Generate a production-ready serverless lakehouse on Amazon S3 using AWS Glue Data Catalog, Apache Iceberg tables, and Amazon Athena: Infrastructure-as-Code (Terraform preferred; CloudFormation acceptable) plus Athena DDL/DML, with cost guardrails, compaction, and schema evolution built in from day one.

Target configuration (use unless I override):
- Region: [AWS_REGION]
- Data bucket: [DATA_S3_BUCKET]
- Glue database: [DATABASE_NAME]
- Partition columns: [PARTITION_COLUMNS]
- Monthly Athena query budget: [MONTHLY_QUERY_BUDGET]

## Verify Reality FIRST (before generating anything)
1. Use the AWS Documentation MCP tools (aws_read_documentation, aws_search_documentation) to CONFIRM, not assume: Athena engine v3, the per-query and workgroup data-usage controls, exact OPTIMIZE/VACUUM syntax, and every Iceberg table property you set (write_target_data_file_size_bytes, vacuum_max_snapshot_age_seconds, format-version).
2. Confirm Athena, Glue, and Lake Formation are available in [AWS_REGION].
3. If anything cannot be verified, say so and use the safe documented form. NEVER invent a table property, CLI flag, or API name.
4. State every assumption about my source data in one short block before any code.

## Detailed Requirements

### 1. Storage & Catalog
- Bucket layout: s3://[DATA_S3_BUCKET]/raw/ (landing), /iceberg/[DATABASE_NAME]/ (analytics), /athena-results/ (output).
- Glue database [DATABASE_NAME]; crawler over /raw/ at cron(0 1 * * ? *), excluding /iceberg/ and /athena-results/.
- Analytics table: Apache Iceberg, format-version 2, Parquet, write_target_data_file_size_bytes='134217728' (128 MB).
- Register the location with Lake Formation; grant least-privilege table/column permissions to the named query role. Do NOT rely on IAMAllowedPrincipals.

### 2. Partitioning + Compaction (mandatory)
- Partition by [PARTITION_COLUMNS]: day-granularity date plus at most one low-cardinality column; justify in one sentence and warn against high-cardinality keys.
- CTAS that reads the raw crawled table and writes the partitioned Iceberg table.
- Compaction recipe: OPTIMIZE <table> REWRITE DATA USING BIN_PACK toward 128 MB; ALTER TABLE ... SET TBLPROPERTIES ('vacuum_max_snapshot_age_seconds'='259200'); VACUUM to expire snapshots older than 3 days and remove orphan files (may need multiple runs on large tables).
- Scheduled runner via EventBridge Scheduler invoking athena:StartQueryExecution — OPTIMIZE nightly cron(0 3 * * ? *), VACUUM weekly cron(0 4 ? * SUN *).

### 3. Athena Cost-Per-Query Guardrails (mandatory)
- Dedicated workgroup: EnforceWorkGroupConfiguration=true, engine version 3, result location s3://[DATA_S3_BUCKET]/athena-results/ with SSE-S3 or SSE-KMS.
- BytesScannedCutoffPerQuery=53687091200 (50 GB): any query exceeding it is auto-canceled. State the valid range (10 MB to 7 EB), that cancel is the only per-query action, and that canceled queries bill only bytes scanned — worst case ~$0.25 per query.
- Workgroup-wide data-usage alert to an SNS topic at 80% of ([MONTHLY_QUERY_BUDGET] / 30 / $5-per-TB) daily; it notifies but does not cancel.
- AWS Budgets monthly budget filtered to Athena at [MONTHLY_QUERY_BUDGET], alerts at 80% and 100%.

### 4. Schema-Evolution Recipe (mandatory)
- Generate ALTER TABLE ADD COLUMNS, RENAME COLUMN, and DROP COLUMN examples; explain that Iceberg tracks columns by ID, so adds/renames rewrite no data and existing queries keep working.
- Contrast with Hive/crawler tables, where one new producer field returns NULLs or throws HIVE_PARTITION_SCHEMA_MISMATCH.
- One-line read-back query proving a new column returns correctly on old and new partitions.

### 5. Security & Cost
- Least-privilege IAM: Glue service role scoped to this database and bucket; query role scoped to the workgroup, results prefix, and table.
- S3: Block Public Access, default encryption, TLS-only bucket policy, lifecycle moving /raw/ to Glacier Instant Retrieval at 90 days.
- Monthly cost-estimate table (Athena scans, Glue DPU-hours, S3, catalog requests) inside [MONTHLY_QUERY_BUDGET].

## Error Management
Give Cause -> Fix for: HIVE_PARTITION_SCHEMA_MISMATCH after drift on the raw table; the crawler re-cataloging the Iceberg location as plain Hive (exclude /iceberg/); OPTIMIZE failing on concurrent writes (retry off-peak); the cap canceling a legitimate backfill (use a separate admin workgroup — never raise the default); VACUUM not finishing in one pass (rerun until clean).

## Verification & Acceptance (emit as proof)
- An "Acceptance Checklist" of copy-paste CLI commands: aws athena get-work-group (BytesScannedCutoffPerQuery=53687091200), aws glue get-table (parameter table_type=ICEBERG), aws lakeformation list-permissions, aws budgets describe-budgets, aws scheduler get-schedule.
- A deliberately unbounded SELECT * with the expected result stated: canceled with "Bytes scanned limit was exceeded."
- A partition-pruned query plus aws athena get-query-execution showing Statistics.DataScannedInBytes covers only the selected partitions.

## Output Format
Produce, in order: (1) assumptions, (2) Terraform (or CloudFormation), (3) Athena DDL/DML, (4) EventBridge Scheduler resources, (5) cost-estimate table, (6) Acceptance Checklist. No preamble. The result must deploy in under 15 minutes, and it must be impossible to run a query that scans more than 50 GB by accident.
```

## What You Get

- A Terraform module (or CloudFormation template): Glue database, scheduled crawler, Lake Formation grants, Iceberg table, the Athena workgroup with the 50 GB per-query cap, SNS data-usage alert, and AWS Budgets at 80%/100%.
- Athena DDL for the Iceberg table (format-version 2, 128 MB target) plus a CTAS migration that partitions raw data.
- Maintenance SQL—`OPTIMIZE ... REWRITE DATA USING BIN_PACK`, `ALTER TABLE ... SET TBLPROPERTIES`, `VACUUM`—wired to EventBridge Scheduler nightly/weekly.
- A schema-evolution playbook (`ADD COLUMNS`/`RENAME COLUMN`/`DROP COLUMN`) with a read-back test query.
- Least-privilege IAM roles, an S3 lifecycle policy, a monthly cost-estimate table, and an Acceptance Checklist of CLI commands proving every guardrail.

## Example Output

Assumptions: newline-delimited JSON in s3://acme-data/raw/events/, ~40 GB/day, filtered mostly by event_date and region.

```sql
CREATE TABLE lakehouse_prod.events (
  event_id string, event_type string, region string,
  event_ts timestamp, event_date date
)
PARTITIONED BY (event_date, region)
LOCATION 's3://acme-data/iceberg/lakehouse_prod/events/'
TBLPROPERTIES ('table_type'='ICEBERG','format'='PARQUET','format-version'='2',
  'write_target_data_file_size_bytes'='134217728');

OPTIMIZE lakehouse_prod.events REWRITE DATA USING BIN_PACK;
```

Acceptance: `aws athena get-work-group --work-group lakehouse-prod --query "WorkGroup.Configuration.BytesScannedCutoffPerQuery"` returns `53687091200` (50 GB). Running `SELECT * FROM events` with no partition filter is canceled with "Bytes scanned limit was exceeded" — guardrail confirmed.

## AWS Services Used

AWS Glue (Data Catalog, crawlers, ETL jobs), Amazon Athena (workgroups, engine v3, Iceberg), Apache Iceberg, Amazon S3, AWS Lake Formation, Amazon EventBridge Scheduler, Amazon SNS, AWS Budgets, AWS IAM, AWS KMS.

## Well-Architected Alignment

- Cost Optimization: 50 GB per-query auto-cancel, SNS alert at 80% of daily budget, AWS Budgets at 80%/100%, partition pruning, 128 MB compaction, Glacier Instant Retrieval lifecycle.
- Operational Excellence: IaC-defined, scheduled crawler and OPTIMIZE/VACUUM jobs, an Acceptance Checklist that emits CLI proof, a Cause -> Fix runbook.
- Security: Lake Formation least-privilege grants (no IAMAllowedPrincipals), scoped Glue and query roles, S3 Block Public Access, TLS-only bucket policy, SSE-S3/SSE-KMS.
- Reliability: Iceberg format-version 2 ACID snapshots, in-place schema evolution, 3-day snapshot retention for time-travel rollback after a bad write.
- Performance Efficiency: partition design guidance, 128 MB target files, Athena engine v3 with Iceberg statistics and Parquet column pruning.

## Cost Notes

Athena is $5.00 per TB scanned (us-east-1, on-demand) with a 10 MB per-query minimum and free DDL; canceled queries bill only bytes scanned, so the 50 GB cap holds any single query to ~$0.25. Glue ETL bills $0.44 per DPU-hour (1-minute minimum on Glue 2.0+); crawlers bill the same rate with a 10-minute minimum per run. The Glue Data Catalog free tier (first million objects, first million requests monthly) covers this stack. S3 Standard is ~$0.023/GB-month; Glacier Instant Retrieval ~$0.004/GB-month. A realistic starter (40 GB/day landed, pruned dashboards scanning ~5 GB/query) lands well under $150/month—versus ~$1,920/day for an unpartitioned 4 TB dashboard query refreshing every 15 minutes.

## Troubleshooting

- Query canceled with "Bytes scanned limit was exceeded." Cause: the scan exceeded the 50 GB cap, usually a missing partition filter. Fix: add a predicate on a partition column (e.g. `WHERE event_date = current_date`); run intentional large scans in a separate admin workgroup.
- HIVE_PARTITION_SCHEMA_MISMATCH or NULL columns after a producer added a field. Cause: Hive/crawler tables cannot evolve schema safely. Fix: `ALTER TABLE ... ADD COLUMNS` on the Iceberg table; columns are tracked by ID, so old and new partitions both read correctly.
- Queries slow (60-90s) and over-scanning. Cause: thousands of tiny files. Fix: run `OPTIMIZE <table> REWRITE DATA USING BIN_PACK` toward 128 MB, then confirm the lower data-scanned figure.
- S3 storage creeping up despite compaction. Cause: expired snapshots and orphan files. Fix: set `vacuum_max_snapshot_age_seconds=259200` and run `VACUUM` weekly; rerun large tables until clean.
- "Insufficient Lake Formation permission" on a valid query. Cause: the role lacks the table/column grant. Fix: `aws lakeformation grant-permissions` for SELECT on that table to the role.
