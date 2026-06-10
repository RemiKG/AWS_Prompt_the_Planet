# CUR 2.0 + Athena Cost Warehouse: 10 Production SQL Queries for Per-Team, Per-Feature, and Unit-Economics Answers

Stand up a queryable cost warehouse from your raw AWS bill—CUR 2.0 Data Exports, one Athena table, and 10 battle-tested SQL queries—so "what does feature X cost per customer?" takes 30 seconds, not a week of spreadsheet archaeology.

## The Problem

Your bill hits $38,000/month and the questions start: "What does the search team actually spend?" "What's the cost per 1,000 orders?" "Why is non-prod 35% of the bill?" Cost Explorer can't answer them—it groups by a limited set of dimensions, hides untagged spend inside tag-filtered views, and its API bills $0.01 per request the moment you script against it. Cost Anomaly Detection tells you something *changed*; it cannot tell you what anything *costs per unit of business*. Meanwhile the complete answer already exists: the Cost and Usage Report contains every line item, every resource ID, every tag, every effective rate—millions of rows per month that nobody at your startup can query. Teams ship features with zero cost feedback, finance builds a fragile spreadsheet from CSV exports, and the one engineer who once set up "the old CUR" left. This is a data problem, not a detection problem: you need your bill in SQL.

## Who This Is For

Startup platform engineers and founding engineers who own the AWS bill, FinOps-curious CTOs at seed-to-Series-B companies ($5K–$500K/month spend), and data engineers asked to wire cost into business dashboards. You should be comfortable running CLI commands and reading SQL; you do not need prior CUR experience—the legacy CUR's worst traps (sparse, ever-changing columns) are exactly what CUR 2.0's fixed 125-column schema fixes.

## How to Use

1. Open Kiro CLI (or Claude Code / Cursor / Amazon Q Developer CLI) in an empty working directory with AWS credentials for your management (payer) account loaded.
2. Copy the System Prompt below into the session. Replace every bracketed placeholder: `[MANAGEMENT_ACCOUNT_ID]`, `[BUCKET]`, `[EXPORT_NAME]`, monthly budget, and your real cost allocation tag keys.
3. Let the assistant run the Reality Checks first (it will use the AWS CLI and the AWS Documentation MCP server). If no CUR 2.0 export exists yet, it generates the creation commands—approve them, then expect up to 24 hours before the first Parquet files land.
4. Once data exists, the assistant deploys the AWS-delivered `crawler-cfn.yml` (or the delivered `create-table.sql`), creates the `cost-analytics` Athena workgroup, and emits all 10 queries plus `verify.sh`.
5. Run `bash verify.sh`, confirm every check is PASS, then save the 10 queries as Athena saved queries and (optionally) wire the QuickSight module.

**Prerequisites**

- **Required Access:** Management/payer account (or a member account—then you only see that account's data). IAM permissions: `bcm-data-exports:CreateExport/GetExport/ListExports`, `ce:ListCostAllocationTags`, `ce:UpdateCostAllocationTagsStatus`, `athena:*` on workgroup `cost-analytics`, `glue:CreateDatabase/CreateTable/GetTable/StartCrawler`, `cloudformation:CreateStack`, and S3 admin on the export bucket.
- **Recommended Background:** Basic SQL (Athena is Trino-based—`UNNEST`, map access with `col['key']`), and how user-defined cost allocation tags are activated in the Billing console.
- **Tools Required:** AWS CLI v2; Kiro CLI or any agentic assistant; AWS Documentation MCP server (`aws_search_documentation`, `aws_read_documentation`) so the assistant verifies CUR 2.0 column names against the official table dictionary instead of guessing.

**Key Parameters:** Export name (`cur2-analytics`), granularity (HOURLY), format (Parquet, overwrite mode), resource IDs (TRUE), split cost allocation (TRUE), Glue DB/table (`cur2_db.cur2`), workgroup scan cap (10 GB/query), Athena results expiry (30 days), CUR retention (25 months), tag keys (`user_team`/`user_feature`/`user_environment`), unit-metric table (`business_metrics`), monthly budget for q10 ($40,000).

**Troubleshooting:** If all 10 queries return zero rows on day one, that's expected—Data Exports takes up to 24 hours to deliver a new export's first refresh. Confirm `.snappy.parquet` objects exist under `s3://<bucket>/<prefix>/<export-name>/data/BILLING_PERIOD=YYYY-MM/` before touching any SQL.

## System Prompt

```
# AWS Cost Analytics Warehouse — CUR 2.0 + Athena Build Request

Act as a senior FinOps data engineer. Build me a production-ready cost analytics warehouse that turns my raw AWS billing data into a SQL-queryable Athena table and ships 10 ready-made queries answering per-team, per-feature, and unit-economics questions. This is a data platform, not an anomaly detector.

## Project Overview
Management account [MANAGEMENT_ACCOUNT_ID], home region us-east-1, monthly spend roughly [$40,000], cost allocation tags in use: [Team, Feature, Environment]. Daily business metrics (active customers, orders) are available as CSV.

## Reality Checks — Run These BEFORE Generating Anything
1. `aws bcm-data-exports list-exports --region us-east-1` — if a CUR 2.0 export already exists, reuse it; never create a duplicate.
2. `aws s3 ls s3://[BUCKET]/cur2/[EXPORT_NAME]/data/` — confirm `BILLING_PERIOD=YYYY-MM` partitions with `.snappy.parquet` objects exist before writing any DDL. A new export takes up to 24 hours to deliver; if empty, say so and stop instead of debugging phantom failures.
3. `aws ce list-cost-allocation-tags --status Active` — confirm the tag keys are ACTIVE. If not, emit the activation steps plus a 12-month backfill request, and mark q02/q03/q05/q06 as pending until data arrives.
4. Discover the real tag keys with `SELECT DISTINCT t.key FROM cur2_db.cur2 CROSS JOIN UNNEST(map_keys(resource_tags)) AS t(key)` — never hardcode `user_team` without proof it exists.
5. The CUR 2.0 schema is fixed at 125 columns. If a column is not present in the AWS-delivered `[EXPORT_NAME]-create-table.sql`, it does not exist — do not invent columns. Verify any uncertain column against the official CUR 2.0 table dictionary via aws_read_documentation.
6. Execute every generated query with `LIMIT 10` in workgroup `cost-analytics` and report bytes scanned before declaring it done.

## Detailed Requirements

### 1. Data Foundation (AWS Data Exports)
- CUR 2.0 standard export `cur2-analytics`: TIME_GRANULARITY=HOURLY, INCLUDE_RESOURCES=TRUE, INCLUDE_SPLIT_COST_ALLOCATION_DATA=TRUE, Parquet, overwrite mode.
- S3 bucket `[company]-cur2-[ACCOUNT_ID]`: Block Public Access on, SSE-S3, bucket policy granting `bcm-data-exports.amazonaws.com` and `billingreports.amazonaws.com` write access scoped with `aws:SourceAccount`, lifecycle rule expiring objects after 25 months.

### 2. Athena Layer
- Glue database `cur2_db`, table `cur2` partitioned by `billing_period` (string, 'YYYY-MM'). Prefer deploying the AWS-delivered `crawler-cfn.yml` from the export prefix; fall back to the delivered `create-table.sql` plus `ALTER TABLE ... ADD PARTITION` per billing period.
- Athena workgroup `cost-analytics`: EnforceWorkGroupConfiguration=true, BytesScannedCutoffPerQuery=10737418240 (10 GB), results to `s3://[BUCKET]/athena-results/` with 30-day expiry.

### 3. The 10 Queries (`queries/q01.sql`–`q10.sql`)
Every query MUST filter on `billing_period` for partition pruning, use the amortized-cost expression (SavingsPlanCoveredUsage → `savings_plan_savings_plan_effective_cost`; DiscountedUsage → `reservation_effective_cost`; exclude `SavingsPlanNegation`, `SavingsPlanRecurringFee`, `RIFee`), and exclude `Tax`/`Credit`/`Refund` rows except where noted:
1. **q01 Month-over-month by service** — top 20 by `line_item_product_code` with MoM delta and percent, 6-month window.
2. **q02 Cost per team** — `resource_tags['user_team']` with an explicit `(untagged)` bucket, percent of total, and MoM change.
3. **q03 Cost per feature** — `resource_tags['user_feature']` within a chosen team, top 15.
4. **q04 Tag coverage scorecard** — percent of amortized spend missing each tag key, by service and `line_item_usage_account_name`.
5. **q05 Unit economics** — join daily tagged cost to `business_metrics` (DDL included, CSV at `s3://[BUCKET]/business-metrics/`) → cost per active customer and per 1,000 orders, daily, trailing 30 days.
6. **q06 Cost per environment** — prod/staging/dev by team; flag any team whose non-prod share exceeds 30%.
7. **q07 Top 30 resources** — by `line_item_resource_id` with owning team and 7-day trend.
8. **q08 Data-transfer breakdown** — `line_item_usage_type LIKE '%DataTransfer%'` by service and usage type, with GB and $/GB.
9. **q09 Commitment realized savings** — effective cost vs `pricing_public_on_demand_cost` per service; reporting only, no purchase recommendations.
10. **q10 Month-end forecast** — MTD amortized spend, daily run rate, linear month-end projection vs the [$40,000] budget with a RED/AMBER/GREEN verdict; include a bill-reconciliation variant that keeps Tax/Credit/Refund.

### 4. Cost Guardrails
Entire stack must run for ≤ $3/month at Athena's $5/TB-scanned pricing. Each query must scan < 1 GB; the header comment of every .sql file must state the question it answers, expected bytes scanned, and expected cost.

### 5. QuickSight Module (optional, emit last)
Athena-backed SPICE dataset over q02 + q10, daily refresh at 08:00 UTC, 4 visuals: spend by team, MoM by service, tag coverage, forecast vs budget. Note pricing in comments: $24/author/month, $3/reader/month.

## Deliverables
1. `setup.sh` — idempotent AWS CLI commands (export, bucket + policy, workgroup, crawler stack).
2. `queries/q01.sql … q10.sql` with the required header comments.
3. `business_metrics.ddl` plus a 5-row sample CSV.
4. `README.md` — data dictionary for the 12 CUR 2.0 columns used, tag-activation and backfill runbook.
5. `verify.sh` — the acceptance evidence script.

## Acceptance Evidence
`verify.sh` must print: partition count, row count for the current `billing_period`, q01's total reconciled within 2% of the Billing console month-to-date figure, bytes scanned per query, and PASS/FAIL per check. Do not declare success until all 10 queries execute with zero errors.

Hands-on deploy time under 30 minutes; first answer within 24 hours of export creation; every query returns in under 10 seconds. Output the files directly, without any preamble.
```

## What You Get

- `setup.sh` — idempotent CLI script: CUR 2.0 export creation, hardened S3 bucket with the `bcm-data-exports.amazonaws.com` bucket policy, `cost-analytics` Athena workgroup with a 10 GB per-query scan cap, and the `crawler-cfn.yml` stack deployment.
- `queries/q01.sql`–`q10.sql` — the 10 production queries, each with a header stating the business question, expected scan size, and cost.
- `business_metrics.ddl` + sample CSV — the join table that powers unit economics (cost per customer, cost per 1,000 orders).
- `README.md` — a 12-column CUR 2.0 data dictionary (the only columns you actually need out of 125) plus the cost-allocation-tag activation and 12-month backfill runbook.
- `verify.sh` — acceptance script proving the warehouse reconciles within 2% of the Billing console.
- Optional QuickSight module — SPICE dataset definitions and a 4-visual executive dashboard spec.

## Example Output

```
$ bash verify.sh
[PASS] partitions found: 6 (2026-01 .. 2026-06)
[PASS] rows in billing_period=2026-05: 4,182,907
[PASS] q01 total $39,412.88 vs console MTD $39,407.15 (delta 0.01%)

-- q02_cost_per_team.sql · billing_period='2026-05' · 214.6 MB scanned · $0.0011
team         amortized_cost   pct_of_total   mom_change
platform        18,442.17        46.8%         +6.3%
search           8,310.55        21.1%         -2.1%
growth           5,102.40        12.9%        +11.8%
(untagged)       4,887.02        12.4%            --
```

## AWS Services Used

AWS Data Exports (Cost and Usage Report 2.0), Amazon Athena, Amazon S3, AWS Glue (Data Catalog + crawler), AWS CloudFormation, AWS Billing and Cost Management (cost allocation tags), Amazon QuickSight (optional), AWS IAM.

## Well-Architected Alignment

- **Cost Optimization** — the entire kit operationalizes COST03 ("monitor cost and usage"): amortized-cost accounting, tag-coverage scorecard, unit economics, and a forecast-vs-budget verdict; the Athena workgroup's 10 GB scan cap keeps the analytics layer itself from becoming a cost problem.
- **Operational Excellence** — `verify.sh` reconciles the warehouse against the Billing console within 2% before anyone trusts a number; every query is versioned SQL in git, not console clicks.
- **Security** — Block Public Access, SSE-S3, and a service-principal bucket policy scoped by `aws:SourceAccount`; billing data never leaves your account.
- **Performance Efficiency** — Parquet + partition pruning on `billing_period` keeps 4-million-row months under 10-second, sub-cent queries.
- **Reliability** — overwrite-mode delivery plus the delivered `Manifest.json` makes ingestion self-healing across the up-to-two-week late corrections AWS applies to a closed billing period.

## Cost Notes

- CUR 2.0 export itself: **$0** — AWS delivers it free.
- S3 storage: hourly granularity with resource IDs for a ~$40K/month bill ≈ 1–3 GB Parquet/month → **~$0.05/month** at $0.023/GB (us-east-1), ~$1.40/month once 25 months accumulate.
- Athena: $5/TB scanned, 10 MB minimum per query; a partition-pruned monthly query scans 50–500 MB → **$0.0003–$0.0025 per question**. Running all 10 queries daily ≈ $0.75/month. The 10 GB workgroup cutoff caps a runaway query at $0.05.
- Glue: crawler at $0.44/DPU-hour with a 10-minute minimum ≈ $0.07/run; a daily schedule ≈ **$2.20/month** (drop to monthly runs—new partitions only appear once a month—for ~$0.07/month). Data Catalog free tier covers this table.
- QuickSight (optional): $24/author/month + $3/reader/month; SPICE beyond the included capacity at $0.38/GB-month.
- **Core stack total: under $3/month** against a $38,000/month bill—0.008% overhead for complete cost visibility.

## Troubleshooting

1. **`resource_tags['user_team']` returns NULL everywhere.** Cause: the tag was never activated as a *cost allocation tag*, or was activated after the usage occurred. Fix: Billing console → Cost allocation tags → activate `Team`, then request a backfill (up to 12 months); allow 24 hours, and remember CUR 2.0 lower-cases keys to `user_team`.
2. **Crawler created dozens of junk tables or the table schema is wrong.** Cause: the crawler was pointed at the export root, so it ingested `metadata/` and `execution_status/` folders. Fix: use the AWS-delivered `crawler-cfn.yml` (already scoped to `data/`), or set the crawler's include path to `s3://<bucket>/<prefix>/<export-name>/data/` only.
3. **Numbers are 5–15% higher than Cost Explorer.** Cause: you're summing raw `line_item_unblended_cost` including `Tax`, `Fee`, and double-counting SP/RI rows. Fix: use the kit's amortized expression and line-item-type filters; compare against Cost Explorer with "amortized costs" selected.
4. **Query fails with `Query exhausted resources` or hits the scan cap.** Cause: missing `billing_period` predicate forces a full-table scan across all months. Fix: every query must filter `billing_period = 'YYYY-MM'` (or a 6-month list); confirm with the bytes-scanned figure Athena reports.
5. **Last month's totals changed after the month closed.** Cause: AWS legitimately refreshes a closed billing period for up to two weeks (final fees, support charges, credits). Fix: treat a billing period as final only after the 15th of the following month; `verify.sh` re-reconciles on demand.
