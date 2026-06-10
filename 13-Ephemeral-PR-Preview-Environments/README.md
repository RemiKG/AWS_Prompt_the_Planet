# Ephemeral Per-PR Preview Environments with TTL Auto-Destroy

Spin up a full-stack preview per pull request and guarantee its destruction on PR close or by a nightly reaper—so reviewers click a live URL instead of your team paying for forgotten stacks.

## The Problem

Per-PR preview environments are the best code-review tool nobody runs safely. The deploy half is easy; the expensive half is what happens next. A PR is force-merged and the close webhook is missed, the destroy job fails on a non-empty S3 bucket, or a branch is abandoned—and the stack lives forever. One preview with an Application Load Balancer (~$16.20/mo), a `db.t3.micro` RDS instance (~$12/mo), a NAT Gateway ($32.40/mo plus data), and a Route 53 record burns roughly $60/month doing nothing. Ten orphans is $600/month of pure waste—and without per-env budgets, one runaway job adds hundreds more before anyone looks.

This prompt generates a production-ready system: teardown guaranteed by three independent mechanisms, a budget alarm and hard cost cap on every environment, and the live URL commented straight back onto the PR.

## Who This Is For

- Startup teams (2-30 engineers) who want a clickable URL per PR instead of "pull the branch and run it locally."
- Platform engineers on GitHub Actions with CDK or Terraform who need the guardrails they keep meaning to build.
- Anyone burned by an orphaned stack who wants destruction to be the default, not a hope.

## How to Use

1. Open your infrastructure repo in Kiro CLI or Claude Code with credentials for your sandbox AWS account.
2. Pick your IaC tool (CDK TypeScript or Terraform workspaces) and base domain, then paste the System Prompt below, replacing every `[BRACKETED]` placeholder.
3. Let it verify reality first (OIDC provider, hosted zone, wildcard ACM cert) and report gaps before writing any code.
4. Review the workflows, stack, and reaper, then open a throwaway PR to watch the full open → deploy → comment → close → destroy loop.

Prerequisites:
- Required Access: An AWS sandbox account with rights to create IAM roles, VPC, RDS, ECS/Fargate, ALB, Route 53 records, AWS Budgets, and EventBridge Scheduler; a GitHub repo with Actions enabled and admin rights to add an OIDC trust.
- Recommended Background: GitHub Actions YAML; CDK (TypeScript) or Terraform HCL; basic Route 53 and ACM.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`); AWS CLI v2; GitHub CLI (`gh`).

Key Parameters: TTL (72h), reaper (cron(0 3 * * ? *), 03:00 UTC), per-env budget ($30/mo, 80%/100% actual alerts, tag-filtered), hard cap ($45/env via Cost Explorer), subdomain (pr-<number>.preview.example.com), DB (db.t3.micro Postgres 16, single-AZ), log retention (7 days), preview cap (10), Fargate scale-to-zero after 24h idle.

Troubleshooting: If destroy fails with "DELETE_FAILED: bucket is not empty," that is expected—objects block stack deletion. The stack prevents it with `autoDeleteObjects: true` (CDK) or `force_destroy = true` (Terraform); otherwise empty objects and delete markers, then re-run destroy.

## System Prompt

```
# Ephemeral Per-PR Preview Environment System — Build Request

You are a senior platform engineer. Generate a PRODUCTION-READY system that provisions a full-stack preview environment per GitHub pull request and GUARANTEES its destruction. Write no code until Verify Reality is complete and findings are reported.

## Project Context
- IaC tool: [CDK_TYPESCRIPT or TERRAFORM_WORKSPACES]
- Base domain (existing Route 53 public hosted zone): [preview.example.com]
- Account/region: non-production sandbox, [us-east-1]
- App shape: [one container on ECS Fargate behind an ALB + RDS Postgres db.t3.micro]
- Source repo: [github.com/ORG/REPO]

## Verify Reality FIRST
1. Via the AWS Documentation MCP server (aws_read_documentation, aws_search_documentation), confirm current resource types, flags, and constraints for GitHub OIDC federation, Route 53 alias records, ACM DNS validation, AWS Budgets, and EventBridge Scheduler cron syntax. Never invent a flag.
2. Ask me to run and paste: aws sts get-caller-identity; aws route53 list-hosted-zones; aws iam list-open-id-connect-providers (confirm token.actions.githubusercontent.com); gh auth status.
3. Confirm a wildcard ACM certificate for *.[preview.example.com] is ISSUED in the region; if missing, STOP and give the exact aws acm request-certificate --validation-method DNS command and validation-CNAME step.
4. List every gap with its remediation command. Do not proceed past a missing OIDC provider or hosted zone.

## Requirements
1. Deploy on pull_request [opened, reopened, synchronize] with concurrency group preview-pr-${{ github.event.number }} and cancel-in-progress: true. OIDC ONLY (aws-actions/configure-aws-credentials, role-to-assume); no long-lived keys. Per PR: stack/workspace preview-pr-<number>, subdomain pr-<number>.[preview.example.com] as a Route 53 A-alias to the ALB. Tag EVERY resource: PrNumber, ManagedBy=preview-system, ExpiresAt (UTC ISO-8601 = now + [72h]), CostCenter=preview; remind me to activate both as cost allocation tags.
2. Comment back: ONE sticky comment (actions/github-script, find-and-update via a hidden <!-- preview-env --> marker) with the HTTPS URL, ExpiresAt, and budget; requires permissions: pull-requests: write. On failure, comment the failed step and run link.
3. Teardown by THREE independent mechanisms: (a) on pull_request [closed], empty versioned buckets, destroy the stack/workspace, delete the Route 53 record and budget; (b) nightly reaper — EventBridge Scheduler cron(0 3 * * ? *) → Lambda that lists environments via the Resource Groups Tagging API and destroys any past ExpiresAt OR whose PR is closed/merged, catching missed webhooks and force-merges; (c) refuse the 11th concurrent preview (default cap 10). Destroy is idempotent (exit 0 if already gone). Bake destroyability in: CDK removalPolicy DESTROY + autoDeleteObjects: true, deletionProtection: false on RDS; Terraform force_destroy = true, skip_final_snapshot = true, deletion_protection = false.
4. Cost guardrails: one AWS Budget per preview ($30/mo) filtered by the PrNumber tag, SNS alerts at 80%/100% of ACTUAL cost (Budgets refreshes up to 3x/day — a backstop, not real-time); reaper-enforced $45/env hard cap via a tag-filtered Cost Explorer GetCostAndUsage check; smallest Fargate task, db.t3.micro single-AZ, 7-day logs, ONE shared NAT Gateway (never one per env — $32.40/mo each).
5. Security: OIDC trust restricts sub to repo:ORG/REPO:*; permission policy scoped to the preview namespace, tag-conditioned — show both JSON documents. RDS credentials in Secrets Manager. ALB ingress 443 only; DB reachable only from the app security group.

## Error Handling (Cause → Resolution, encoded in the code)
- DELETE_FAILED, bucket not empty → empty objects AND delete markers; set the destroy-safe flags.
- sts:AssumeRoleWithWebIdentity denied → OIDC provider missing or sub mismatch; print the expected sub string.
- Destroy hangs on an RDS final snapshot → verify the skip flags.
- Budget reads $0 forever → cost allocation tags not activated, or under 24h old.
- Racing synchronize events → the concurrency group serializes them; never remove it.

## Deliverables (each a separate, labeled file)
1. .github/workflows/preview-deploy.yml — deploy + sticky comment. 2. preview-destroy.yml — teardown. 3. Reaper (EventBridge Scheduler + Lambda). 4. Per-PR stack with tagging and destroy-safe flags. 5. OIDC trust + permission policy JSON. 6. Versioned-bucket-empty script. 7. README with lifecycle diagram and Key Parameters. 8. Cost table: per preview + waste avoided at 10 orphans.

## Verification & Acceptance
Give the exact gh pr create test, the expected sticky comment, and aws resourcegroupstaggingapi get-resources --tag-filters Key=ManagedBy,Values=preview-system proving ZERO leftovers after close. Add a reaper drill: strand one environment, wait one cycle, prove it is gone. Acceptance bar: live HTTPS URL within 8 minutes of opening a PR; zero tagged resources after close AND after the drill. Orphaned spend is not reduced — it is structurally impossible.
```

## What You Get

- `preview-deploy.yml` — OIDC deploy in a concurrency group, Route 53 A-alias, tag-filtered budget, sticky URL comment.
- `preview-destroy.yml` — close/merge teardown of the stack, DNS record, and budget.
- Reaper — EventBridge Scheduler + Lambda killing environments past ExpiresAt, with a closed PR, or over the $45 cap.
- Per-PR stack (CDK or Terraform workspace module) with full tagging and destroy-safe flags; OIDC trust and least-privilege policy scoped to repo:ORG/REPO:*; versioned-bucket-empty script.
- README with lifecycle diagram, Key Parameters, and a `gh pr create` test; cost-estimate table with monthly waste avoided.

## Example Output

(excerpt) Reality check complete. Found: OIDC provider token.actions.githubusercontent.com (present), hosted zone preview.example.com (present), wildcard ACM cert *.preview.example.com (MISSING). STOP: request it with `aws acm request-certificate --validation-method DNS` and add the returned CNAME. ... Generated both workflows, the reaper, and the $30 tag-filtered budget. Sticky comment: "Preview ready: https://pr-482.preview.example.com (expires 2026-06-13T15:00Z, budget $30/mo)."

## AWS Services Used

AWS Identity and Access Management (IAM OIDC federation), Amazon Route 53, AWS Certificate Manager (ACM), Amazon ECS on AWS Fargate, Elastic Load Balancing (Application Load Balancer), Amazon RDS for PostgreSQL, Amazon VPC (NAT Gateway), AWS Secrets Manager, AWS Budgets, AWS Cost Explorer, Amazon CloudWatch, Amazon EventBridge Scheduler, AWS Lambda, Amazon SNS, AWS CloudFormation / AWS CDK or Terraform, Amazon S3, AWS Resource Groups Tagging API.

## Well-Architected Alignment

- Cost Optimization: TTL auto-destroy plus nightly reaper guarantee zero orphaned spend; one shared NAT Gateway; db.t3.micro single-AZ; 7-day logs; $30 tag-filtered budget; $45 hard cap; cap of 10 previews.
- Operational Excellence: full lifecycle in GitHub Actions; sticky URL comment-back; idempotent destroy; the reaper drill proves the safety net; all IaC.
- Security: OIDC short-lived credentials only, trust scoped to repo:ORG/REPO:*, Secrets Manager for DB credentials, HTTPS-only ALB.
- Reliability: three independent teardown mechanisms; a missed webhook never leaves a stack running; concurrency groups prevent racing deploys.
- Sustainability: resources exist only during active review; idle services scale to zero.

## Cost Notes

A single active preview runs roughly: ALB ~$16.20/mo, db.t3.micro Postgres ~$12/mo, Fargate small task ~$10/mo, shared NAT Gateway ~$32.40/mo split across all previews; Route 53 records negligible; cost budgets and ACM public certificates free; hosted zone $0.50/mo. Activate the PrNumber and CostCenter cost allocation tags—up to 24 hours to appear in billing data. The real return is waste avoided: ten orphans at ~$60/mo is ~$600/mo reclaimed. With the 72h TTL and $30 budget, a single preview's lifetime cost stays under ~$5.

## Troubleshooting

- Cause: PR force-merged, close webhook never fired. Fix: the nightly reaper reconciles by tag and PR state and destroys it anyway—no webhook required.
- Cause: destroy fails with "bucket is not empty" / DELETE_FAILED. Fix: the stack bakes in the destroy-safe flags; for older stacks, empty objects and delete markers, then re-run the idempotent destroy.
- Cause: "Not authorized to perform sts:AssumeRoleWithWebIdentity." Fix: confirm the OIDC provider exists and the trust sub condition matches repo:ORG/REPO:* exactly.
- Cause: TLS error on the preview URL. Fix: the wildcard ACM certificate must be ISSUED and DNS-validated in the ALB's region before deploy.
- Cause: budget reads $0 or alerts arrive hours late. Fix: activate the cost allocation tags (up to 24h to appear); Budgets refreshes up to 3x/day — the TTL reaper and $45 cap enforce the real ceiling.
