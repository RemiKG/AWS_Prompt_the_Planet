# Full-Stack TypeScript on Amplify Gen 2: Cognito Auth, AppSync Data, and Per-Branch Environments Without Hand-Wired Glue

One typed codebase defines auth, data, and hosting with isolated environments per Git branch—ship a production-ready product backend this week instead of hand-wiring Cognito, AppSync, DynamoDB, and IAM for a month.

## The Problem

Standing up a "simple" product backend by hand means coordinating six services: a Cognito user pool, an identity pool, IAM roles, an AppSync GraphQL API, one DynamoDB table per model, and CI/CD. Done in the console, that is 30–60 hours of clicking and copy-pasted ARNs, and the result drifts the moment someone edits dev but not prod. The predictable failures: an authorization rule that exposes user data through a public API key, and a backend nobody can reproduce because it exists only in one engineer's account.

AWS Amplify Gen 2 collapses all of it into **four TypeScript files totaling roughly 120 lines**, deployed to a personal cloud sandbox in about 5 minutes, with full type inference from schema to React components. Teams stall on three questions this prompt answers concretely: exactly which files to write, what Amplify abstracts versus what you still own, and what it costs at 1,000 users (~$11/month; near $0 on the free tier).

## Who This Is For

- Founders and full-stack TypeScript developers with a React, Next.js, or Vue frontend who need real auth and persistence
- Pre-seed teams with no platform engineer who still need isolated dev/staging/prod environments
- Gen 1 Amplify users confused by the CLI-to-code shift (Gen 2 has no `amplify push`; everything is `defineBackend` in code)
- NOT for join-heavy relational schemas (use Aurora) or teams needing raw CDK control of every resource on day one

## How to Use

1. **Claude Code / Kiro CLI / Cursor:** open your frontend repo (or an empty folder) and paste the System Prompt below.
2. Replace the bracketed placeholders: `[APP_NAME]`, `[REGION]` (default us-east-1), `[FRAMEWORK]`, and the data model list (default `Todo`, `Profile`).
3. Enable the AWS Documentation MCP server so the assistant verifies current Gen 2 APIs instead of hallucinating Gen 1 syntax.
4. Review the generated `amplify/` directory, then run `npx ampx sandbox` to deploy your personal cloud sandbox; it hot-reloads on save.
5. Connect your Git repository in the Amplify console; every branch gets its own isolated fullstack environment. Merge to `main` to ship production.

**Prerequisites**

- **Required Access:** an AWS account able to create Cognito, AppSync, DynamoDB, CloudFormation, and IAM resources; for branch deploys, an Amplify service role with the `AmplifyBackendDeployFullAccess` managed policy
- **Recommended Background:** TypeScript and basic React; no Gen 1 Amplify experience needed (or wanted—the APIs differ)
- **Tools Required:** Node.js 18.16+, npm 9+, AWS CLI v2 with a configured profile, Git, an AI assistant with the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`)

**Key Parameters:** APP_NAME, REGION (us-east-1), data models (Todo + Profile), password minimum (12 chars), defaultAuthorizationMode (userPool), auth rule (allow.owner()), PITR (enabled), budget ($25/mo, 80%/100% alerts), sandbox identifier (one per developer)

**Troubleshooting:** If the first `npx ampx sandbox` run fails with "This AWS account and region has not been bootstrapped," that is expected on a fresh account—Gen 2 sandboxes deploy via CloudFormation/CDK and need a one-time `npx cdk bootstrap aws://<ACCOUNT_ID>/<REGION>`.

## System Prompt

```
# Full-Stack TypeScript Backend on Amplify Gen 2 — Build Request

## Project Overview
Scaffold a production-ready full-stack TypeScript application on AWS Amplify Gen 2 for [APP_NAME]: Amazon Cognito email authentication, a typed AWS AppSync GraphQL API backed by Amazon DynamoDB, and Amplify Hosting with per-branch environments. Frontend: [React + Vite | Next.js 14+]. Region: [us-east-1]. Data models: [Todo, Profile].

## Verify Reality First (before writing any code)
1. Run `node -v` and `npm -v`. Require Node.js >= 18.16 and npm >= 9; stop and report if older.
2. Run `aws sts get-caller-identity` to confirm credentials, and confirm the target region supports Amplify Hosting.
3. Check the latest versions of `aws-amplify`, `@aws-amplify/backend`, and `@aws-amplify/backend-cli` on npm and pin them. Verify `defineAuth` and `defineData` signatures against current Amplify Gen 2 docs (aws_read_documentation). NEVER emit Gen 1 syntax (`amplify init`, `amplify push`, `aws-exports.js`).
4. If an `amplify/` directory already exists, read it and extend it — never overwrite.

## Detailed Requirements

### 1. Backend Definition — exact file set
- `amplify/backend.ts` — `defineBackend({ auth, data })`; add a CDK override enabling DynamoDB point-in-time recovery on every model table via `backend.data.resources.cfnResources.amplifyDynamoDbTables`.
- `amplify/auth/resource.ts` — `defineAuth` with email login, 12-character minimum password, email account recovery.
- `amplify/data/resource.ts` — `defineData` with typed `a.schema(...)` models, `defaultAuthorizationMode: 'userPool'`, `allow.owner()` on every model. NEVER place `allow.publicApiKey()` on user data — hard stop. Export `type Schema = ClientSchema<typeof schema>`.
- `amplify/package.json` and `amplify/tsconfig.json` matching `npm create amplify@latest` output.

### 2. Frontend wiring
- `src/lib/amplifyClient.ts`: `Amplify.configure(outputs)` from `amplify_outputs.json` (gitignored), plus a typed client from `generateClient<Schema>()`.
- An auth flow using the Amplify UI React `<Authenticator>` with sign-up, sign-in, sign-out.

### 3. Environments and deployment
- Per-developer cloud sandboxes: `npx ampx sandbox` (isolated per developer identifier, hot-reloads on save; `npx ampx sandbox delete` tears down everything).
- Per-branch environments: an `amplify.yml` whose backend phase runs `npx ampx pipeline-deploy --branch $AWS_BRANCH --app-id $AWS_APP_ID`. `main` is production; every connected Git branch gets its own isolated backend stack.
- Secrets via `npx ampx sandbox secret set` locally and Amplify console secrets for branches — never in source.

### 4. Cost guardrails
Everything serverless and on-demand; target under $15/month at 1,000 MAU (Cognito Essentials free to 10,000 MAU; AppSync $4.00/M operations; DynamoDB on-demand $0.625/M writes, $0.125/M reads; Hosting $0.01/build-minute, $0.15/GB served). Include an AWS Budget at $25/month with 80% and 100% alerts.

### 5. Honest ownership table
Produce a two-column table: what Amplify Gen 2 manages (Cognito pools, AppSync API and resolvers, DynamoDB tables and GSIs, generated IAM roles, build/CDN/TLS) versus what I still own (per-model authorization rules, Amazon SES email beyond Cognito's 50-email/day default sender, backup policy beyond PITR, custom domain DNS, CloudWatch alarms, access-pattern design at scale, schema migrations).

## Deliverables Requested
1. The complete file set above, full TypeScript contents, no unresolved placeholders.
2. The `amplify.yml` build spec.
3. A README runbook: sandbox → branch → production promotion, secret handling.
4. The ownership table from requirement 5.
5. A monthly cost worksheet at 100 / 1,000 / 10,000 MAU.

## Error Management
- Sandbox fails "has not been bootstrapped": Cause — no CDKToolkit stack in this account/region. Resolution — `npx cdk bootstrap aws://<ACCOUNT_ID>/<REGION>`, rerun.
- Frontend throws "Amplify has not been configured": Cause — `amplify_outputs.json` missing. Resolution — keep the sandbox running, or `npx ampx generate outputs --app-id <APP_ID> --branch main`.

## Acceptance Evidence (emit this, not just code)
Output a verification checklist proving the build works: `npx tsc --noEmit` passes; `npx ampx sandbox` reaches deployed status in under 10 minutes; two test users each create a record and CANNOT read each other's (verified with actual queries); a `staging` branch deploys its own isolated stack.

Align with the AWS Well-Architected Framework (Security, Reliability, Cost Optimization, Operational Excellence). Output complete file contents directly, without any preamble.
```

## What You Get

1. `amplify/backend.ts` — backend wiring plus a CDK override enabling DynamoDB point-in-time recovery
2. `amplify/auth/resource.ts` — Cognito email auth, 12-char minimum password, email recovery
3. `amplify/data/resource.ts` — typed schema, `userPool` default auth mode, `allow.owner()` on every model
4. `amplify/package.json` + `amplify/tsconfig.json` — backend workspace files
5. `amplify.yml` — branch pipeline running `npx ampx pipeline-deploy`
6. `src/lib/amplifyClient.ts` — `Amplify.configure(outputs)` + `generateClient<Schema>()`, plus an `<Authenticator>` entry point
7. README runbook — sandbox → branch → production promotion, secrets, teardown
8. The honest ownership table — what Amplify abstracts vs what you still own
9. Cost worksheet at 100 / 1,000 / 10,000 MAU
10. Acceptance evidence — type-check, sandbox deploy timing, two-user owner-isolation test

## Example Output

> Verified Node 20.11.1 / npm 10.2.4; pinned aws-amplify 6.x and @aws-amplify/backend 1.x. Generated 4 backend files (118 lines). Sandbox deployed in 4m 38s. Owner isolation: user-a created Todo t-001; user-b's list returned 0 items — PASS. PITR on. Cost at 1,000 MAU: Cognito $0.00, AppSync $6.00, DynamoDB $0.53, Hosting $4.22 — ≈ $10.75/month.

## AWS Services Used

AWS Amplify (Gen 2 backend tooling + Hosting), Amazon Cognito, AWS AppSync, Amazon DynamoDB, AWS CloudFormation, AWS IAM, Amazon CloudFront (Hosting CDN), AWS Budgets, Amazon CloudWatch

## Well-Architected Alignment

- **Operational Excellence:** the whole backend is code in Git; per-developer sandboxes and per-branch stacks make every environment reproducible and disposable
- **Security:** Cognito user-pool auth by default, `allow.owner()` row-level isolation, generated least-privilege IAM roles, secrets out of source, `amplify_outputs.json` gitignored
- **Reliability:** managed multi-AZ services throughout; DynamoDB point-in-time recovery enabled by CDK override rather than left at default
- **Performance Efficiency:** AppSync and DynamoDB on-demand scale to zero and to spikes without capacity planning
- **Cost Optimization:** every component is pay-per-request; a $25 budget with 80%/100% alerts ships with the kit; idle environments cost ~$0

## Cost Notes

Small app at **1,000 MAU** (us-east-1, 1.5M API ops, 600K writes / 1.2M reads, 100 builds × 3 min, 8 GB served):

- Amazon Cognito (Essentials tier): first 10,000 MAU free, then $0.015/MAU → **$0.00**
- AWS AppSync: $4.00 per million query/mutation operations → **$6.00**
- Amazon DynamoDB on-demand: $0.625/M writes + $0.125/M reads; 25 GB storage always free → **$0.53**
- Amplify Hosting: $0.01/build-minute ($3.00) + $0.15/GB served ($1.20) + $0.023/GB-month stored ($0.02) → **$4.22**
- **Total ≈ $10.75/month.** Accounts created before July 15, 2025 keep the legacy 12-month free tier (1,000 build minutes, 15 GB served, 5 GB stored monthly), pushing this near $0; newer accounts draw Free Tier credits.
- Every extra sandbox or branch environment idles at roughly **$0.00** — everything bills per request, not per hour.

## Troubleshooting

1. **`ampx sandbox`: "account and region has not been bootstrapped"** — Cause: no CDKToolkit stack yet. Fix: `npx cdk bootstrap aws://<ACCOUNT_ID>/<REGION>` once, rerun.
2. **Frontend throws "Amplify has not been configured"** — Cause: `amplify_outputs.json` missing. Fix: keep `npx ampx sandbox` running, or `npx ampx generate outputs --app-id <APP_ID> --branch main`.
3. **Sign-up emails stop arriving** — Cause: Cognito's default sender caps at 50 emails/day per account. Fix: configure Amazon SES as the Cognito email sender before launch — it sits in the "you still own" column.
4. **Branch deploy fails at the backend phase** — Cause: `amplify.yml` lacks the `pipeline-deploy` backend phase, or the service role is missing `AmplifyBackendDeployFullAccess`. Fix: add the phase, attach the managed policy.
5. **Another user can read my records** — Cause: a model carries `allow.publicApiKey()` or lacks `allow.owner()`. Fix: set `defaultAuthorizationMode: 'userPool'` and `allow.owner()` on every user-data model, redeploy, rerun the two-user isolation test.

**Bar to clear:** from empty folder to a deployed, type-safe, owner-isolated backend in your own cloud sandbox in under 10 minutes — and a production branch the same day.
