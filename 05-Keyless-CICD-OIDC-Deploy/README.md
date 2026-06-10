# Keyless GitHub Actions to AWS: OIDC Trust Roles That Kill Long-Lived Access Keys

Generate the IAM OIDC provider, multi-environment trust roles, and production-ready plan-gate-then-apply workflows for GitHub Actions deploys to AWS with zero stored secrets — a leaked token expires in an hour instead of draining your account.

## The Problem

The most common AWS credential leak in startups is not a sophisticated attack — it is `AKIA...` access keys pasted into GitHub repository secrets. Bots assume leaked keys within minutes of a public commit; AWS maintains the `AWSCompromisedKeyQuarantine` managed policies and a GitHub secret-scanning partnership because it happens at scale.

The blast-radius math is brutal. BEFORE OIDC: a static key pair in repo secrets works from any IP, any machine, until someone rotates it — typically never. AFTER OIDC: GitHub Actions exchanges a signed identity token for temporary credentials via `sts:AssumeRoleWithWebIdentity` that live ~1 hour, are pinned to one repository and branch by the trust policy `sub` claim, and cannot be replayed. Most teams never switch because the condition keys are fiddly and one wrong wildcard silently trusts every repo on GitHub.

## Who This Is For

Startup engineers and platform teams still deploying from GitHub Actions with `aws_access_key_id` in repository secrets. Anyone wiring CI/CD for a new service who wants the keyless path from day one. Founders preparing for a SOC 2 review where "no long-lived cloud credentials in CI" is a control.

## How to Use

1. Open your AWS-deploying repository in Kiro CLI or Claude Code with the AWS Documentation MCP server connected.
2. Paste the System Prompt below into a new chat or save it as `.kiro/steering/oidc-cicd.md`.
3. Replace the bracketed placeholders: `[GITHUB_ORG]`, `[REPO_NAME]`, `[AWS_ACCOUNT_ID]`, `[AWS_REGION]`, and the environment list (default dev/staging/prod).
4. Let the assistant verify account state, then review the generated Terraform, trust policies, and workflows.
5. Run `terraform apply`, commit the workflows, then push a no-op change to a non-prod branch to confirm the plan job assumes the role with no stored secrets.

Prerequisites:
- Required Access: An IAM principal allowed to run `iam:CreateOpenIDConnectProvider`, `iam:CreateRole`, `iam:PutRolePolicy`, and `iam:CreatePolicy` in each target account.
- Recommended Background: Basic Terraform or AWS CLI familiarity; GitHub Actions workflow YAML and the `permissions` block.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`); Terraform 1.5+ or AWS CLI v2; the `aws-actions/configure-aws-credentials` action.

Key Parameters: provider URL (token.actions.githubusercontent.com), audience (sts.amazonaws.com), MaxSessionDuration (3600s default, 43200s max), role-duration-seconds (for long applies), prod branch (main), plan-gate (required reviewers on prod), thumbprint (omit — AWS uses trusted-CA verification).

Troubleshooting: If the assume-role step fails with "Not authorized to perform sts:AssumeRoleWithWebIdentity", the most common cause is a missing `permissions: id-token: write` block — without it GitHub never mints the OIDC token at all.

## System Prompt

```
# GitHub Actions to AWS Keyless CI/CD — Infrastructure Request

## Objective
Produce production-ready, copy-paste artifacts that let GitHub Actions deploy to AWS via OIDC federation with NO long-lived access keys anywhere. Repository: [GITHUB_ORG]/[REPO_NAME]. Account: [AWS_ACCOUNT_ID]. Region: [AWS_REGION]. Environments: dev, staging, prod.

## Verify Reality FIRST (before generating anything)
1. Confirm the active account matches [AWS_ACCOUNT_ID] via `aws sts get-caller-identity`; on mismatch, STOP and report.
2. Run `aws iam list-open-id-connect-providers`; if `token.actions.githubusercontent.com` is registered, REUSE its ARN — duplicates fail.
3. Confirm via the AWS Documentation MCP server (aws_read_documentation) the CURRENT sub-claim format, trust condition keys, and latest major tag of `aws-actions/configure-aws-credentials`. Never invent a version.
4. State findings before writing IaC; use documented safe defaults for anything unverifiable, and say so.
5. Ask the user: single or multi-account (repeat provider + roles per account if multi)? Which services does this repo deploy (drives the prod policy)? Prod branch and Environment name (defaults: main, prod)?

## Hard Rules (RFC 2119)
- You WILL NEVER output `aws_access_key_id`, `aws_secret_access_key`, or the credentials action configured with static keys. This is a hard stop.
- You WILL scope every trust policy with BOTH an `aud` AND a `sub` condition. Omitting `sub` trusts every repository on GitHub — a critical defect.
- You WILL grant least privilege per environment: dev MAY use a broader managed policy; prod MUST use a scoped customer-managed policy, never `AdministratorAccess`.
- You WILL set the audience to exactly `sts.amazonaws.com`.

## Required Artifacts
1. Terraform for ONE `aws_iam_openid_connect_provider` with url `https://token.actions.githubusercontent.com` and client_id_list `["sts.amazonaws.com"]`. Omit the thumbprint — AWS verifies the JWKS TLS certificate against its trusted root CAs; note the fallback only as a comment.
2. Four `aws_iam_role` resources with `max_session_duration = 3600` and trust statements using Action `sts:AssumeRoleWithWebIdentity`, Principal = the provider ARN, `StringEquals` on `token.actions.githubusercontent.com:aud` = `sts.amazonaws.com`, and `StringEquals` (NOT StringLike) on `token.actions.githubusercontent.com:sub` with these EXACT values:
   - dev:     repo:[GITHUB_ORG]/[REPO_NAME]:ref:refs/heads/dev
   - staging: repo:[GITHUB_ORG]/[REPO_NAME]:ref:refs/heads/staging
   - prod:    repo:[GITHUB_ORG]/[REPO_NAME]:environment:prod
   - plan-only (PR jobs): repo:[GITHUB_ORG]/[REPO_NAME]:pull_request
3. Two workflows in `.github/workflows/`: `plan.yml` (pull_request trigger, plan-only role, posts `terraform plan` as a PR comment, NO apply) and `apply.yml` (push to dev/staging/main; prod job bound to the `prod` GitHub Environment, gated by required reviewers). Each MUST declare `permissions: { id-token: write, contents: read }` and use `aws-actions/configure-aws-credentials` pinned to the verified tag with `role-to-assume`, `aws-region`, and `role-session-name`. NO access-key inputs. This is the plan-gate-then-apply flow.
4. A least-privilege customer-managed policy for the prod role scoped to the services this repo deploys — never `*`.
5. `BLAST-RADIUS.md` comparing static keys vs OIDC: lifetime (forever vs ~3600s), scope (any caller vs one repo+branch), revocation (manual rotation vs automatic expiry), detection (CloudTrail `AssumeRoleWithWebIdentity` events).

## Verification & Acceptance (emit this — prove it works)
- `aws iam get-role --role-name <prod-role>` shows `MaxSessionDuration: 3600` and the exact `sub` condition.
- A negative test: a run from a non-trusted branch MUST fail with `Not authorized to perform sts:AssumeRoleWithWebIdentity` — that denial proves the scoping works.
- `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity --max-results 5` showing the GitHub identity and zero access-key events.
- An IAM Access Analyzer check confirming no role is publicly assumable.

## Error Handling (Cause -> Resolution)
- "EntityAlreadyExists" on provider create -> URL already registered; data-source the existing ARN, do not recreate.
- assume-role fails on a valid branch -> `sub` mismatch or missing `id-token: write`; print the token sub and diff against the trust policy.
- prod applies without approval -> Environment protection unset; require reviewers on `prod`.
- credentials expire mid-apply -> 1-hour default session; set `role-duration-seconds` and raise `max_session_duration` (up to 43200s).

## Cost & Output
OIDC federation and STS calls are free; the only cost is optional CloudTrail data events. Output every artifact as a separate fenced code block with its file path, no preamble; end with the Acceptance section. Deployable in under 10 minutes: one provider, four roles, two workflows, zero stored secrets. A pipeline with no keys has nothing to leak and nothing to rotate.
```

## What You Get

- `oidc-provider.tf` — one `aws_iam_openid_connect_provider` (audience `sts.amazonaws.com`, thumbprint omitted with explanatory comment).
- `iam-roles.tf` — three environment roles plus a PR plan-only role, each with scoped `assume_role_policy` and `max_session_duration = 3600`.
- `prod-policy.json` — a least-privilege customer-managed policy (no `*`, no `AdministratorAccess`).
- `.github/workflows/plan.yml` — pull_request-triggered plan job with `id-token: write`, posts the plan to the PR.
- `.github/workflows/apply.yml` — apply job with prod gated behind a required reviewer.
- `BLAST-RADIUS.md` — before/after leak-impact table.
- Acceptance checks: `aws iam get-role`, a CloudTrail lookup, the negative-test denial, an IAM Access Analyzer check.

## Example Output

The generated prod trust condition:

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:acme/checkout-service:environment:prod"
  }
}
```

The `apply.yml` job declares `permissions: id-token: write, contents: read` and calls `aws-actions/configure-aws-credentials` with `role-to-assume`, `aws-region`, and `role-session-name` — no access-key inputs anywhere.

## AWS Services Used

AWS IAM (OIDC identity provider, roles, customer-managed policies), AWS STS (`AssumeRoleWithWebIdentity`), AWS CloudTrail (role-assumption audit), IAM Access Analyzer (public-assumability check), AWS Budgets (spend guardrail). Deployed via Terraform; GitHub Actions integrates through `aws-actions/configure-aws-credentials`.

## Well-Architected Alignment

- Security: Eliminates long-lived credentials — the framework's top identity recommendation. Short-lived STS tokens, least-privilege roles, and `sub`/`aud` scoping enforce branch-scoped access; CloudTrail and IAM Access Analyzer supply the audit trail.
- Operational Excellence: Plan-gate-then-apply makes every change reviewable and reproducible; prod approval is a controlled change gate.
- Reliability: Per-environment roles isolate blast radius so a dev misconfiguration cannot touch prod.
- Cost Optimization: OIDC and STS are free; a recommended AWS Budget caps the damage of any residual compromise.

## Cost Notes

The OIDC path itself is $0 — IAM OIDC providers, roles, and `sts:AssumeRoleWithWebIdentity` calls carry no charge. The only incremental cost is optional CloudTrail data events at about $0.10 per 100,000 events. The cost avoided: one leaked admin key can run up a five-figure bill in a day. An AWS Budgets alarm at 80% and 100% of a $500/month threshold is cheap insurance.

## Troubleshooting

- "Not authorized to perform sts:AssumeRoleWithWebIdentity" on a legitimate branch -> the workflow is missing `permissions: id-token: write`, so no OIDC token is minted. Add the block and re-run.
- Same error with a token present -> the `sub` string does not match the run's subject. Print the token sub and align the `StringEquals` condition exactly — never a wildcard.
- "EntityAlreadyExists" creating the provider -> one already exists for `token.actions.githubusercontent.com`. Reference its ARN as a Terraform data source.
- A fork or unauthorized repo can assume the role -> the `sub` condition was left as a broad wildcard. Pin it to the exact `repo:ORG/REPO:...` value per environment and confirm with IAM Access Analyzer.
- prod apply runs with no approval -> the GitHub `prod` Environment has no required reviewers. Enable them so `apply.yml` blocks until approved.
- Credentials expire during a long apply -> the default 1-hour session ended. Set `role-duration-seconds` and raise `max_session_duration` (up to 43200s).
