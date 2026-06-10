# Savings Plans & RI Commitment Analyzer: Stop Over-Committing on AWS Compute

Turn Cost Explorer, the CUR, and Compute Optimizer into a laddered, no-regrets commitment plan—so you capture 40-60% compute savings without stranding a single dollar on idle reservations.

## The Problem

Teams either ignore Savings Plans and pay 100% On-Demand, or they panic-buy a 3-year All-Upfront Compute Savings Plan sized to last month's spend—then refactor, migrate to Graviton, or lose a customer, and watch utilization collapse on a non-cancellable contract. A $40/hr commitment running at 65% utilization burns $14 every hour—$122,600 a year—and AWS does not refund it.

Every account already holds the data: Cost Explorer exposes `GetSavingsPlansPurchaseRecommendation` and the `StartCommitmentPurchaseAnalysis` what-if API; CUR 2.0 holds hourly line items; Compute Optimizer flags rightsizing to do *before* committing. This prompt ties them into one rule: **commit to the floor, not the forecast.** Buy Savings Plans only up to your stable trough—the hourly spend you run regardless—leave the spiky top on On-Demand or Spot, and ladder small commitments quarterly as the floor proves itself.

## Who This Is For

- Startup founders and solo platform engineers staring at a 5-figure monthly compute bill with zero commitments.
- FinOps practitioners who need a defensible purchase memo for a CFO before signing a multi-year contract.
- Anyone burned by an over-bought RI or Savings Plan who wants a repeatable, conservative methodology, not a gut call.

## How to Use

1. Enable Cost Explorer (Billing console); first refresh takes up to 24 hours. Enable a daily CUR 2.0 export to S3—hourly granularity in Cost Explorer covers only 14 days, so the CUR proves the 60-day floor.
2. Confirm at least 30 days of usage history (60 preferred); the analyzer refuses to recommend on less.
3. Open Kiro CLI, Claude Code, or any AI assistant with the AWS Documentation MCP server plus an AWS API MCP or AWS CLI v2 profile.
4. Paste the System Prompt. Provide payer account ID (or `PAYER` scope), home region, existing commitments, and planned 12-month migrations.
5. Review the floor, ladder, and memo, then buy manually in the Savings Plans console—nothing is purchased automatically.

Prerequisites:
- Required Access: read-only billing—`AWSBillingReadOnlyAccess` plus `ce:Get*`, `ce:StartSavingsPlansPurchaseRecommendationGeneration`, `ce:StartCommitmentPurchaseAnalysis`, `compute-optimizer:Get*`, `cur:DescribeReportDefinitions`. No write or purchase permission needed.
- Recommended Background: Compute Savings Plans (flexible, up to ~66% off) vs. EC2 Instance Savings Plans (family+region locked, up to ~72% off); Standard vs. Convertible RIs.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`); AWS CLI v2 or AWS API MCP execution path.

Key Parameters: utilization target >=95% (reduce if projected <90%), coverage 70-80% (never 100%), LookbackPeriodInDays `SIXTY_DAYS` (min `THIRTY_DAYS`), TermInYears `ONE_YEAR` for rung 1 (`THREE_YEARS` only on a 2-quarter proven floor), PaymentOption `NO_UPFRONT` (`PARTIAL_UPFRONT` if discount delta >3%), ladder step 25% of remaining gap per quarter, floor = P5 of trailing-60-day hourly On-Demand spend.

Troubleshooting: An empty details array from `GetSavingsPlansPurchaseRecommendation` is expected when recommendations were never generated—run `StartSavingsPlansPurchaseRecommendationGeneration` and wait up to 24 hours.

## System Prompt

```
# Savings Plans & RI Purchase Analysis — Requirements Document

You are a senior AWS FinOps analyst. Produce a conservative, defensible, production-ready compute-commitment purchase plan for my account. The governing rule is non-negotiable: COMMIT TO THE FLOOR, NOT THE FORECAST. Recommend commitments only up to the stable trough I am certain to run; leave spiky demand on On-Demand or Spot.

## Step 0 — Verify Reality FIRST

You WILL NOT invent numbers. Before any analysis:
1. Confirm Cost Explorer is enabled with >=30 days of data; fewer is a hard STOP—tell me to gather more history.
2. Verify operation names with aws_read_documentation before calling. The real Cost Explorer operations: GetCostAndUsage, GetSavingsPlansPurchaseRecommendation, GetSavingsPlansUtilization, GetSavingsPlansCoverage, GetReservationPurchaseRecommendation, GetReservationUtilization, GetReservationCoverage, GetRightsizingRecommendation, StartSavingsPlansPurchaseRecommendationGeneration, StartCommitmentPurchaseAnalysis, GetCommitmentPurchaseAnalysis. Never call anything unconfirmed.
3. If GetSavingsPlansPurchaseRecommendation returns no details, run StartSavingsPlansPurchaseRecommendationGeneration, state it can take up to 24 hours, and pause—never fabricate.
4. Cost Explorer HOURLY granularity covers only the last 14 days; derive the 60-day hourly floor from the CUR 2.0 export via Athena, or combine the DAILY 60-day shape with the 14-day HOURLY window and state the approximation.
5. Echo back AccountScope, LookbackPeriodInDays, TermInYears, PaymentOption, and the data window before proceeding.

## Required Information (ask if missing)

Payer account ID and AccountScope (PAYER or LINKED); home region and monthly compute spend; existing Savings Plans / RIs with hourly amounts and expiry; planned 12-month changes (Graviton, EKS, serverless, growth).

## Analysis Requirements

1. RIGHTSIZE FIRST. Pull GetRightsizingRecommendation and Compute Optimizer findings (GetEC2InstanceRecommendations; on OptInRequiredException tell me to opt in—free—and wait up to 24 hours). NEVER commit on top of over-provisioned instances; list every downsize first.
2. ESTABLISH THE NO-REGRETS FLOOR. Compute the P5 of hourly On-Demand compute spend over the trailing 60 days (30 minimum)—the level I run at or above 95% of the time. The first commitment WILL NOT exceed this floor. State it in both On-Demand and SP-rate $/hour.
3. RECOMMEND CONSERVATIVELY. Call GetSavingsPlansPurchaseRecommendation with SavingsPlansType COMPUTE_SP, TermInYears ONE_YEAR, PaymentOption NO_UPFRONT, LookbackPeriodInDays SIXTY_DAYS. PARTIAL_UPFRONT or THREE_YEARS only when the incremental discount exceeds 3% AND the floor has held 2+ quarters. Prefer Compute SP (up to ~66% off; EC2/Fargate/Lambda) over EC2 Instance SP (up to ~72% off; family+region locked) unless the fleet is provably stable. Savings Plans do NOT cover RDS, ElastiCache, OpenSearch, or Redshift—use GetReservationPurchaseRecommendation for those. Never stack on an existing low-utilization plan.
4. SET TARGETS AND LADDER. Coverage 70-80% of eligible compute—NEVER 100%, that strands money in demand troughs. Utilization >=95%; if projected <90%, reduce the commitment. Buy 25% of the remaining floor-to-target gap per quarter so commitments expire on a rolling schedule. Validate every rung with StartCommitmentPurchaseAnalysis / GetCommitmentPurchaseAnalysis before presenting it.
5. COUNT THE COST. The Cost Explorer API bills $0.01 per paginated request—batch efficiently and report call count and cost. Quantify dollars saved AND dollars at risk per rung.

## Error Management (Cause -> Resolution)

- AccessDenied on ce:* -> attach AWSBillingReadOnlyAccess plus ce:Get* and ce:Start*; zero write permissions needed.
- Empty recommendation set -> StartSavingsPlansPurchaseRecommendationGeneration, wait, retry.
- DataUnavailableException -> the window predates available data; shorten and disclose.
- LimitExceededException -> exponential backoff; the API is billed per request.
- Existing commitment under 90% utilization -> no new purchase until expiry.

## Deliverables (produce ALL)

1. commitment-analysis.md — coverage/utilization vs. target, rightsizing list, the floor in $/hr (On-Demand and SP-rate).
2. purchase-ladder.md — quarterly ladder table: rung, type, hourly commitment, term, payment, projected coverage/utilization, dollars saved and at risk.
3. cost-memo.md — CFO-ready one-pager: headline annual saving, break-even utilization, explicit downside if usage drops 20%.
4. verification.md — ACCEPTANCE evidence: every API call with parameters, the data window, raw RecommendationIds and AnalysisIds, and pass/fail checks: commitment <= P5 floor at SP rates; coverage <= 80%; utilization >= 95%; rightsizing first; no overlap with existing commitments.

When in doubt, recommend LESS. Nothing is purchased automatically—output the four documents directly, no preamble, for human review.

Closing rule: a Savings Plan you cannot fully use is more expensive than the On-Demand it replaced. Commit to the floor, not the forecast.
```

## What You Get

- `commitment-analysis.md` — coverage/utilization vs. targets, rightsizing list, the no-regrets floor in $/hour (On-Demand and SP rates).
- `purchase-ladder.md` — quarterly ladder: rung, type, hourly commitment, term, payment, projected coverage/utilization, dollars saved/at risk.
- `cost-memo.md` — CFO-ready one-pager: headline saving, break-even utilization, downside scenario.
- `verification.md` — acceptance evidence: API calls, lookback window, recommendation/analysis IDs, pass/fail checks (floor, coverage <= 80%, utilization >= 95%).

## Example Output

From commitment-analysis.md:
"60-day trailing On-Demand compute: avg $18.40/hr, P5 floor $11.20/hr, peak $31.10/hr. Coverage today: 0%. Compute Optimizer flags 4 over-provisioned m5.2xlarge—downsize to m5.xlarge BEFORE committing; the floor drops to $9.30/hr."

From purchase-ladder.md:
"Rung 1 (now): $6.05/hr Compute SP (the $9.30/hr floor at ~35% SP rates), 1-yr, No Upfront -> covers $9.30/hr of On-Demand, coverage 56%, utilization 97%, ~$28,500/yr saved, $0 at risk. Rung 2 (Q+1): +$1.40/hr after Graviton settles, stepping toward 70-80%."

## AWS Services Used

AWS Cost Explorer and the Cost Explorer API (all eleven commitment operations named in the System Prompt), AWS Cost and Usage Report (CUR 2.0), AWS Compute Optimizer, AWS Savings Plans, EC2 Reserved Instances, AWS Billing and Cost Management, Amazon Athena, IAM.

## Well-Architected Alignment

- Cost Optimization: the core pillar—operationalizes "Adopt a consumption model" by committing only to proven baseline usage and laddering purchases.
- Operational Excellence: every recommendation ships with a verification.md acceptance artifact and a CFO memo—auditable, repeatable.
- Reliability: spiky demand stays on On-Demand/Spot; coverage caps at 70-80%, preserving spike headroom.
- Performance Efficiency: Compute Optimizer rightsizing runs BEFORE any commitment—you never lock in over-provisioned spend.
- Security: read-only billing permissions; purchase access is never granted.

## Cost Notes

- Tooling cost is the Cost Explorer API: $0.01 per paginated request; a full analysis is 15-40 requests = $0.15-$0.40. Compute Optimizer and recommendation generation are free.
- The payoff: Compute SP discounts up to ~66%, EC2 Instance SP up to ~72%. Committing a $9.30/hr post-rightsizing floor at ~35% discount saves ~$28,500 in year one on rung 1 alone—$0 strand risk.
- The avoided downside dwarfs the saving: a $40/hr 3-year commitment at 65% utilization wastes $14/hr—about $122,600 a year—never used, never refunded.

## Troubleshooting

- Empty recommendation array. Cause: never generated for this account. Fix: StartSavingsPlansPurchaseRecommendationGeneration, wait up to 24 hours, retry.
- AccessDenied on ce:*. Cause: missing billing read permissions. Fix: attach AWSBillingReadOnlyAccess plus ce:Get* and ce:Start*.
- Recommendation looks far too large. Cause: sized to recent spend, possibly on over-provisioned instances. Fix: rightsize first, then commit only to the P5 floor at SP rates.
- Floor looks wrong or jumpy. Cause: Cost Explorer hourly granularity covers only 14 days. Fix: enable CUR 2.0 and recompute the P5 from hourly line items in Athena.
- Projected utilization below 90%. Cause: the commitment exceeds your true baseline. Fix: reduce to the P5 floor; ladder the remainder.
