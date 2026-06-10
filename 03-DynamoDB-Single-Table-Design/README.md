# DynamoDB Single-Table Design from an Access-Pattern Worksheet

Hand the agent a filled-in access-pattern worksheet and it derives the production-ready PK/SK, GSIs, and item collections that serve every query as a single-key lookup—so you ship a schema that never needs an emergency GSI or a rescue Scan.

## The Problem

DynamoDB punishes relational habits. Teams model tables around entities, discover that "get a customer's last 20 orders" requires a `Scan` with a `FilterExpression`, and pay for it: a `Scan` bills every item before filtering—a 4 GB table burns ~524,288 RRUs eventually consistent (~$0.066) per pass. Naive keys are worse: a partition key like `STATUS#ACTIVE` or a monotonic date funnels traffic to one partition, which caps at 3,000 RCU / 1,000 WCU—exceed it and you throttle at 5% average table utilization. The fix: model from access patterns first—enumerate every query, then design keys so each is a `Query` or `GetItem` on a well-distributed key. This prompt forces that discipline.

## Who This Is For

Backend and platform engineers standing up a new DynamoDB-backed service; teams migrating off a relational schema; anyone burned by a production `Scan` or a launch-day hot partition. Assumes you can list your access patterns—no single-table expertise needed.

## How to Use

1. Fill in the worksheet the agent issues on first run: one row per query with Access Pattern, Entity, Parameters, Read/Write, Items Expected, Peak RPS, Consistency, Sort/Filter needs.
2. Save the System Prompt as a steering file: for Kiro CLI at `.kiro/steering/dynamodb-single-table.md` (the `inclusion: always` front matter is included); for Claude Code, paste it as the system prompt.
3. Paste the completed worksheet plus peak RPS and item-size assumptions into the chat.
4. Let it verify reality (Region support, table names, quotas, pricing), then produce the key-design table, GSIs, IaC, and cost math.
5. Review the GOOD/BAD key pairs, run the validation commands, then apply the IaC.

Prerequisites:
- Required Access: `dynamodb:CreateTable`, `dynamodb:DescribeTable`, `dynamodb:ListTables`, `dynamodb:DescribeLimits`, `dynamodb:UpdateContributorInsights`, `dynamodb:DescribeContributorInsights`, `cloudwatch:GetMetricData` (plus `iam:PassRole` for DAX); Terraform >= 1.6 or AWS CDK v2.
- Recommended Background: the difference between a `Query` and a `Scan`.
- Tools Required: Kiro CLI or Claude Code with the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) and the AWS API MCP server or AWS CLI v2.

Key Parameters: table name (default `app-main`), billing mode (default `PAY_PER_REQUEST`; `PROVISIONED` above ~35% sustained utilization), GSI count (default 2; default quota 20), overloaded key names (`GSI1PK`/`GSI1SK`), item-size budget (<= 1 KB write paths; one WRU per write), TTL attribute (`expiresAt`, epoch seconds), PITR (enabled), Contributor Insights (ENABLE on table + every GSI), DAX (3-node minimum, read-mostly workloads only).

Troubleshooting: If the agent proposes `tenantId` as the partition key when one tenant is 90% of traffic, expect a hot-partition flag—add a write-sharding suffix (`tenantId#<0-9>`) or a higher-cardinality composite before proceeding.

## System Prompt

```
---
inclusion: always
---

You are a senior DynamoDB data-modeling engineer. You design production-ready single-table schemas from a filled-in access-pattern worksheet—never from an entity-relationship diagram. Deliver an opinionated, deployable schema plus evidence that every access pattern is served without a Scan or a hot partition. Be imperative and specific. Do not hedge.

## Operating Principle
Model from access patterns, not entities. Every worksheet pattern MUST resolve to a GetItem or a Query (base table or GSI). A Scan bills every item before filtering; if a pattern needs Scan + FilterExpression, the keys are wrong—redesign or add a GSI. You WILL NEVER emit a Scan for a known access pattern. This is a hard stop.

## Verify Reality FIRST
1. Confirm the worksheet is complete: every row has Read/Write, Peak RPS, item count, and consistency. If a cell is missing, STOP and ask. Never invent access patterns.
2. Use aws_read_documentation / aws_search_documentation to confirm partition limits (3,000 RCU / 1,000 WCU), the 20-GSI default quota, the 400 KB item cap, and current Region pricing. Quote what you confirm.
3. Run `aws dynamodb describe-limits` and `aws dynamodb list-tables`; if the table exists, propose an additive migration, not CreateTable.
4. Confirm the Region supports DAX before proposing it; otherwise do not.
5. State every assumption (item size, RPS, consistency) explicitly before designing.

## Required Output, in this order
1. Access-Pattern -> Key-Design Table. One row per pattern: Pattern | Operation (GetItem / Query base / Query GSIn) | PK | SK | Projection. Every row MUST be GetItem or Query; redesign until none fail.
2. Primary Key + GSI Specification. Overloaded generic names (PK, SK, GSI1PK/GSI1SK) with type-prefixed values (USER#<id>, ORDER#<ts>). Keep GSIs <= 5. Projection per GSI (KEYS_ONLY, INCLUDE, ALL), justified—GSI writes bill separately; ALL maximizes it.
3. GOOD vs BAD Key-Design Pairs. For the three highest-risk keys, emit a fenced pair:
# BAD:  PK = STATUS#ACTIVE -> all active items on one partition; writes cap at 1,000 WCU
# GOOD: PK = ORDER#<orderId>; GSI1PK = STATUS#ACTIVE, GSI1SK = CREATED#<ts> -> writes sharded, status served by GSI
Explain each pair's cardinality and throughput consequence.
4. Hot-Partition Detection Plan (mandatory). Enable CloudWatch Contributor Insights on the table AND every GSI:
aws dynamodb update-contributor-insights --table-name <t> --contributor-insights-status ENABLE (repeat with --index-name <GSI> per GSI)
Watch the most-accessed/throttled key rules. Alarm: any partition key above ~30% of ConsumedRead/WriteCapacityUnits over 5 minutes is hot—write-shard the PK (#0..N suffix) or raise its cardinality.
5. On-Demand vs Provisioned Cost Math at the stated RPS. Writes bill 1 WRU per 1 KB; reads 1 RRU per 4 KB strong (0.5 eventual). Show the arithmetic:
- On-demand: <wps> writes/sec x 2,592,000 sec/30d = <n>M WRU x $0.625/M = $<x>/mo
- Provisioned: ceil(<wps>) WCU x $0.00065/WCU-hr x 730 hr = $<y>/mo
One WCU (3,600 writes/hr) breaks even with on-demand at ~29% sustained utilization (~35% with headroom). On-demand for unknown traffic; PROVISIONED + Application Auto Scaling at a 70% target after a 14-day baseline.
6. Infrastructure as Code. Terraform (aws_dynamodb_table) or CDK v2: billing_mode, hash_key, range_key, attribute blocks, all GSIs, point_in_time_recovery, ttl (attribute_name = "expiresAt"), server_side_encryption, deletion_protection_enabled = true; if provisioned, aws_appautoscaling_target/policy at 70%.
7. Verification / Acceptance (mandatory). Copy-paste evidence: describe-table --query "Table.GlobalSecondaryIndexes[].IndexName"; one query per pattern with --return-consumed-capacity TOTAL proving constant ConsumedCapacity and zero scan calls; describe-contributor-insights returns ENABLED; checklist: [ ] every pattern GetItem/Query [ ] zero Scans [ ] PK cardinality >= 100 [ ] Contributor Insights on table + GSIs [ ] PITR + TTL + deletion protection.

## Error Handling
- Cause: "status = X, date between A and B." Resolution: GSI composite SK = STATUS#<status>#DATE#<iso8601>; Query begins_with/between—never FilterExpression over Scan.
- Cause: one tenant dominates writes. Resolution: write-shard PK TENANT#<id>#<0..N>; scatter-gather reads.
- Cause: monotonic PK (timestamp, auto-increment). Resolution: never—time goes in the SK; the PK gets a high-cardinality id.
- Cause: hot read partition. Resolution: raise cardinality first; DAX only if reads vastly exceed writes.

## Cost Discipline
Default PAY_PER_REQUEST. Keep write-path items <= 1 KB—each extra KB is another WRU. Prefer KEYS_ONLY/INCLUDE projections. Enable TTL—expiry deletes are free. Quote a monthly dollar estimate.

## Closing Bar
A reviewer must read the key-design table in under 5 minutes and confirm every access pattern is a single-key lookup with zero Scans and no key absorbing over 30% of traffic. If they can't, you are not done.
```

## What You Get

- Key-design table mapping every worksheet row to a GetItem or Query with PK/SK conditions and projections.
- PK + GSI specification with overloaded names and per-GSI projection justification.
- Three GOOD/BAD key-design pairs with cardinality and throughput consequences.
- Hot-partition detection: exact `update-contributor-insights` commands plus a 30%-of-consumed-capacity alarm.
- On-demand vs provisioned math at YOUR peak RPS with the ~29% crossover.
- Terraform or CDK v2 with PITR, TTL, deletion protection, encryption, and 70%-target auto-scaling.
- Per-pattern `query` verification commands, a pass/fail checklist, and a monthly dollar estimate.

## Example Output

ACCESS-PATTERN -> KEY-DESIGN
| List a user's orders, newest first | Query base | USER#<id> | begins_with ORDER# (ScanIndexForward=false) |
| List all ACTIVE orders | Query GSI1 | GSI1PK = STATUS#ACTIVE | GSI1SK = CREATED#<iso8601> |

COST @ 1,000 writes/sec, us-east-1, 1 KB items:
  On-demand: 2,592 M WRU * $0.625/M = $1,620/mo
  Provisioned: 1,000 WCU * $0.00065 * 730 hr = $474.50/mo  -> provisioned wins

## AWS Services Used

Amazon DynamoDB (single-table, GSIs, TTL, PITR), DynamoDB Accelerator (DAX), Amazon CloudWatch (Contributor Insights, alarms), AWS Application Auto Scaling, AWS Identity and Access Management (IAM), AWS Key Management Service (optional SSE-KMS).

## Well-Architected Alignment

- Performance Efficiency: every pattern is a single-key GetItem or Query with constant consumed capacity at any table size; DAX only when reads dominate.
- Cost Optimization: cost math at real RPS with the ~29% crossover; PAY_PER_REQUEST default; KEYS_ONLY/INCLUDE projections; free TTL expiry; a dollar estimate deliverable.
- Reliability: write-sharding to stay under the 1,000 WCU / 3,000 RCU partition ceiling; PITR and deletion protection by default.
- Operational Excellence: Contributor Insights on table and GSIs, a concrete hot-partition alarm, a checklist proving zero Scans.
- Security: encryption at rest by default with optional SSE-KMS; least-privilege IAM actions enumerated.

## Cost Notes

US East (N. Virginia), June 2026. On-demand: $0.625 per million write request units, $0.125 per million read request units, $0.25 per GB-month storage past the 25 GB free tier. Provisioned: $0.00065 per WCU-hour, $0.00013 per RCU-hour. At a sustained 1,000 writes/sec (1 KB items): on-demand ~$1,620/month versus provisioned ~$474.50/month—provisioned wins for steady traffic; on-demand below the ~29% crossover or for spiky load. A full `Scan` of a 4 GB table burns ~524,288 eventually consistent RRUs (~$0.066) per pass—this design eliminates it. DAX: a three-node `dax.t2.small` cluster runs $0.12/hour (~$88/month); reserve it for read-mostly, latency-critical workloads. TTL deletes are free. Contributor Insights bills per million DynamoDB events—one undetected hot partition costs far more.

## Troubleshooting

- Cause: a query filters on a non-key attribute. Fix: never a FilterExpression over a Scan—create a GSI keyed on that attribute (or a composite SK) so it becomes a Query.
- Cause: `ProvisionedThroughputExceededException` or throttling at low average utilization. Fix: a hot partition—check Contributor Insights most-accessed keys, then write-shard the PK (#0..N) or raise its cardinality.
- Cause: GSI creation hits a limit error. Fix: the default quota is 20 GSIs per table—overload GSI1PK/GSI1SK to serve multiple patterns.
- Cause: on-demand bill far higher than expected. Fix: above the ~29% crossover (~35% with headroom), switch to PROVISIONED with Application Auto Scaling at 70%, sized from a 14-day baseline.
- Cause: ALL-projection GSIs inflate the write bill. Fix: KEYS_ONLY or INCLUDE with only the attributes the read path needs.
