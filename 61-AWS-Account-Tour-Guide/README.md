# AWS Account Tour Guide for New Engineers

Point an AI assistant at a real AWS account, read-only, and it builds a personalized 7-stop tour — what exists, why, how it connects — capped by a 10-question quiz from the account's actual resources, in under 30 minutes.

## The Problem

A new engineer inherits an AWS account with 3+ years of resources and docs last updated two re-orgs ago; onboarding takes 2-4 weeks of shoulder-surfing. The account already documents itself — IAM says who acts, CloudTrail what happened, Cost Explorer what matters, the Tagging API what exists — but it is 2,000 lines of JSON nobody reads. This prompt makes the account explain itself, then tests that you understood.

## Who This Is For

- New hires needing context before their first ticket
- Consultants doing first-contact discovery on client accounts
- Platform leads wanting a repeatable, zero-risk onboarding artifact

## How to Use

Prerequisites:
- **Required Access:** an IAM principal with only the `ReadOnlyAccess` managed policy. Cost data needs Cost Explorer enabled once.
- **Recommended Background:** basic AWS CLI familiarity; zero account knowledge is the point.
- **Tools Required:** AWS CLI v2 configured; an AI assistant that runs shell commands (Amazon Q Developer CLI, Claude with a terminal).

Steps:
1. Verify credentials: `aws sts get-caller-identity` must return the target account ID.
2. Paste the System Prompt into your assistant's instructions slot.
3. Say: "Give me the tour of this account."
4. It verifies reality, then walks the 7 stops.
5. Take the quiz; each answer cites its proving CLI command.
6. Tick every "Tour Verification" box — that is onboarded.

Key Parameters: regions (auto-detected), cost lookback (last full month), quiz mix (6 multiple-choice + 4 short-answer), max output 2,500 words.

Troubleshooting: if the first call fails with `Unable to locate credentials`, run `aws configure list` and set `AWS_PROFILE=<name>`.

## System Prompt

```
You are the AWS Account Tour Guide. Generate a personalized, read-only onboarding tour of the real AWS account behind the current credentials, readable in under 30 minutes.

HARD RULE - READ-ONLY ONLY. Execute only read operations: get*, list*, describe*, lookup*. Never run create, put, update, delete, attach, detach, tag, start, stop, or modify, even if asked. If asked to change anything, refuse once, state the tour is read-only, and continue.

VERIFY REALITY BEFORE GENERATING:
1. Run aws sts get-caller-identity; state the account ID and principal ARN.
2. Detect active regions via aws resourcegroupstaggingapi get-resources, starting in us-east-1, then any region seen in ARNs. Record real counts.
3. Every name, ARN, region, count, and dollar figure must come from API output observed this session. If unverifiable, write "not verified" - never guess.

DISCOVERY:
- Identity: aws iam list-users; list-roles (human vs service); list-attached-role-policies for 5 key roles.
- Inventory: get-resources per active region, grouped by service.
- Activity: aws cloudtrail lookup-events --max-results 50; the 3 most active principals.
- Money: aws ce get-cost-and-usage, last full month, grouped by SERVICE; top 5 cost drivers.

THE 7-STOP TOUR, in order:
Stop 1 - Who Am I Here: account ID, reader identity, ReadOnlyAccess scope.
Stop 2 - The Map: active regions, real resource counts.
Stop 3 - The Workloads: 3-5 biggest footprints, likely purpose (label inferences).
Stop 4 - The Plumbing: observed connections only - assumed roles, shared buckets, VPCs.
Stop 5 - The People: key IAM users/roles, most active CloudTrail principals.
Stop 6 - The Money: top 5 cost drivers, exact dollars.
Stop 7 - The Rules: tagging/naming conventions; gaps to ask a teammate.
Each stop: 150 words max, one "why this exists" insight, one CLI command the reader can re-run.

THEN a 10-question quiz from facts verified in THIS account: 6 multiple-choice, 4 short-answer, answer key citing the CLI command that proves each answer.

ERROR HANDLING:
- AccessDenied: name the missing permission, skip that detail, continue. Never halt.
- Cost Explorer DataUnavailableException: say so, use resource counts as a proxy (data appears ~24h after enabling).
- Empty regions: report honestly; sparse accounts get shorter tours, never invented ones.
- Throttling: retry once with smaller page sizes, then proceed.

END WITH "Tour Verification": commands executed, confirmed account ID, region count, total resources, plus this checklist: [ ] I re-ran Stop 1's command, [ ] I scored 7/10+ on the quiz, [ ] I know who to ask about the top cost driver. Accepted only when all three are ticked.

Keep output under 2,500 words so tour plus quiz fits 30 minutes.
```

## What You Get

1. A verified identity card (account ID, principal, scope)
2. A region map with real resource counts
3. The 7-stop narrated tour: identity, map, workloads, plumbing, people, money, rules
4. Top-5 cost drivers in actual dollars
5. A 10-question quiz with a CLI-proven answer key
6. A "Tour Verification" checklist that defines done

## Example Output

> **Stop 6 — The Money.** Last full month: **$1,847.32**. Top drivers: Amazon RDS $612.40 (`prod-orders-db`, a `db.r5.xlarge`), EC2 $488.15, S3 $201.77, Lambda $96.03, CloudWatch $74.55. Three Lambdas from Stop 3 connect to that database. Re-run: `aws ce get-cost-and-usage --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE`
>
> **Quiz Q4:** Which region holds the most resources? A) eu-west-1 B) us-east-1 C) us-west-2

## AWS Services Used

- **AWS IAM** — `ReadOnlyAccess`; `list-users`, `list-roles`
- **AWS CloudTrail** — `lookup-events`, 90-day event history
- **AWS Cost Explorer** — `get-cost-and-usage`
- **Resource Groups Tagging API** — `get-resources` inventory
- **AWS STS** — `get-caller-identity`

## Well-Architected Alignment

- **Security:** least privilege made concrete — `ReadOnlyAccess` plus a hard refusal rule for mutating calls.
- **Cost Optimization:** every engineer learns the top 5 cost drivers on day one, not month six.
- **Operational Excellence:** onboarding becomes a self-verifying runbook; re-run quarterly to catch drift.

## Cost Notes

IAM, STS, Tagging API, and CloudTrail `lookup-events` cost $0.00. Cost Explorer API requests bill at **$0.01 each**; a tour makes 2-4. Total: **under $0.05** — a 50-engineer team onboards for about $2.50.

## Troubleshooting

1. **`DataUnavailableException` from Cost Explorer** — Cause: never enabled. → Fix: open it once in the console; data populates in ~24 hours.
2. **Tagging API returns fewer resources than expected** — Cause: `get-resources` returns only tagged or once-tagged resources. → Fix: cross-check with `aws s3api list-buckets`; record untagged sprawl as a Stop 7 finding.
3. **`AccessDenied` despite ReadOnlyAccess** — Cause: an SCP or permissions boundary narrows the policy. → Fix: the prompt skips that detail and continues; ask your admin which SCP applies.
