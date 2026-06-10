# Local-First AWS Dev with LocalStack + SAM CLI

Run your serverless stack on a laptop and in CI for zero AWS spend — with an honest map of what LocalStack emulates faithfully versus what only real AWS can prove, so a green test suite actually means a green deploy.

## The Problem

Iterating against real AWS is slow and quietly expensive. Every `sam deploy` to test a one-line handler change burns 2-4 minutes of CloudFormation churn, and a team of five running that loop dozens of times a day means hours of dead waiting. Worse, sandbox accounts leak money: a forgotten NAT Gateway is ~$32.40/mo, an idle RDS `db.t3.micro` is ~$12/mo, and a `kinesis` shard left running is ~$10.80/mo. Multiply across every developer's personal scratch stack and "free tier" becomes a few hundred dollars a month of nothing.

So teams reach for LocalStack — and then hit the second trap: treating it as a perfect AWS replica. They mock S3 and DynamoDB happily, then assume IAM denies are enforced (by default they are NOT — you need `ENFORCE_IAM=1`), that RDS or Cognito work on the free Community edition (they are Pro-only), or that Lambda concurrency, KMS grants, and cross-service eventual consistency behave identically. The result is the worst kind of bug: tests pass locally, the PR merges, and it breaks in the AWS account where it actually matters.

This prompt generates a production-ready local-first setup that is honest about its own limits. It produces a LocalStack + SAM CLI dev loop, a per-service endpoint configuration, the exact same integration suite running in a GitHub Actions CI job (CI parity), and a maintained capability table that says plainly what is faithfully emulated and what must be verified against real AWS.

## Who This Is For

- Serverless teams (Lambda + DynamoDB + S3 + SQS/SNS + API Gateway) who want a sub-second inner loop instead of waiting on `sam deploy`.
- Startup engineers tired of paying for personal sandbox stacks and getting bill-shock from forgotten resources.
- Platform/QA engineers who need integration tests that run identically on a laptop and in CI, and who refuse to ship a suite that lies about coverage.

## How to Use

1. Open your serverless repo (with `template.yaml`) in Kiro CLI, Claude Code, or your AI assistant of choice, with Docker running.
2. Paste the System Prompt below, or save it as `.kiro/steering/localstack-sam.md`. Replace every `[BRACKETED]` placeholder (services list, region, runtime).
3. Let the assistant verify reality FIRST — Docker version, LocalStack image tag, SAM CLI version, and which of your services are Community vs. Pro — and report the honest capability table before writing any config.
4. Review the generated `docker-compose.yml`, `samconfig.toml`, endpoint config, test suite, and the GitHub Actions workflow; then run `docker compose up`, `pytest tests/integration`, and watch the same suite pass in CI.

Prerequisites:
- Required Access: Docker Engine 20.10+ (Compose v2) locally. No AWS account is required for the LocalStack path; a real AWS sandbox account is only needed for the optional "real-AWS parity" verification job.
- Recommended Background: Comfort with AWS SAM templates (`template.yaml`), reading a `docker-compose.yml`, and one test runner (pytest with boto3, or jest with the AWS SDK v3).
- Tools Required: AWS Documentation MCP server (`aws___read_documentation`, `aws___search_documentation`) to confirm current SAM CLI flags and service behavior; the LocalStack docs for current Community vs. Pro coverage; AWS CLI v2 (used with `--endpoint-url`); SAM CLI; `awslocal`/`samlocal` wrappers optional.

Key Parameters: LocalStack edition (default Community/free), LocalStack endpoint (http://localhost:4566 — single edge port for all services), SAM Docker network (`--docker-network` shared with the LocalStack container), endpoint-override env (`AWS_ENDPOINT_URL=http://localhost:4566` via `--env-vars env.json`), IAM enforcement (`ENFORCE_IAM=1`, default OFF), DynamoDB billing mode (PAY_PER_REQUEST local), log/test retention (CI artifacts 7 days), CI runner (ubuntu-latest, LocalStack as a service container), parity job cadence (nightly against real AWS, default OFF in PR runs), test timeout (30s per integration test).

Troubleshooting: If `sam local invoke` reaches real AWS instead of LocalStack (and silently charges your account or fails on missing credentials), it is because the Lambda container is not on LocalStack's Docker network and `AWS_ENDPOINT_URL` was not injected — pass `--docker-network` and `--env-vars env.json` so the function resolves http://localhost:4566 from inside the container, not localhost on the host.

## System Prompt

```
# Local-First AWS Development Environment — Requirements

You are a senior platform engineer. Generate a PRODUCTION-READY local-first development and testing setup using LocalStack + AWS SAM CLI, with full CI parity. Do NOT generate any files until you have completed the "Verify Reality" section and printed the honest capability table.

## Project Context
- Serverless services in scope: [LAMBDA, DYNAMODB, S3, SQS, SNS, API_GATEWAY]
- Lambda runtime(s): [python3.12]
- Region used locally and in CI: [us-east-1]
- Test runner: [pytest + boto3] (or jest + AWS SDK v3)
- LocalStack edition assumed: Community (free) unless I tell you otherwise

## Verify Reality FIRST (before writing any file)
1. Confirm tool versions and tell me the exact commands to run: `docker --version` (need 20.10+, Compose v2), `sam --version`, `localstack --version` or the pinned image tag.
2. Using the AWS Documentation MCP (aws___read_documentation, aws___search_documentation) and current LocalStack docs, classify EACH service in scope as: faithfully emulated on Community / Pro-only / partial-or-mocked. Do NOT guess — if a service's local fidelity is uncertain, label it "verify on real AWS".
3. Print an HONEST capability table with three columns: Service | LocalStack support (Community/Pro/partial) | What to still verify on real AWS. State plainly: IAM policy ENFORCEMENT is OFF by default and requires ENFORCE_IAM=1; RDS, Cognito, ECS/EKS, ElastiCache are Pro-only; eventual consistency, throttling/quotas, KMS grant evaluation, and cross-account behavior are NOT reliably reproduced locally.
4. If any in-scope service is Pro-only or only mocked, STOP and tell me before generating tests that would give false confidence.

## Hard Rules
- Use the SINGLE LocalStack edge endpoint http://localhost:4566 for ALL services. Do not invent per-service ports.
- For `sam local invoke`/`start-api`, connect the Lambda container to LocalStack via `--docker-network` and inject `AWS_ENDPOINT_URL=http://localhost:<port>` through `--env-vars env.json`. Inside the SAM/Lambda container the host is the LocalStack container name (e.g. http://localstack:4566), NOT localhost — get this right.
- The SAME integration test suite MUST run locally and in CI with zero code changes — only the endpoint URL differs by env var.
- Pin the LocalStack image to a specific tag (no `:latest`) so CI is reproducible.

## Required Artifacts (produce each as a separate, labeled file)
1. `docker-compose.yml` — LocalStack pinned to a specific tag, port 4566 exposed, SERVICES list, DEBUG=0, plus a commented ENFORCE_IAM=1 line.
2. `env.json` — SAM `--env-vars` file setting AWS_ENDPOINT_URL and dummy AWS_ACCESS_KEY_ID/SECRET (LocalStack accepts any) and AWS_REGION.
3. A `Makefile` or `taskfile` with targets: `up`, `down`, `bootstrap` (create tables/buckets/queues via awslocal or CLI --endpoint-url), `invoke`, `test`.
4. Bootstrap script using `aws --endpoint-url=http://localhost:4566 ...` (or awslocal) to create the DynamoDB table (PAY_PER_REQUEST), S3 bucket, and SQS queue the app needs.
5. The integration test suite (pytest+boto3 or jest+SDK v3) that points boto3/SDK at the AWS_ENDPOINT_URL env var and exercises a real round-trip (write to DynamoDB, drop an S3 object, send+receive an SQS message, invoke the Lambda).
6. `.github/workflows/ci.yml` — runs LocalStack as a service container (pinned tag), starts the stack, runs bootstrap, then runs the IDENTICAL test suite; uploads logs as a 7-day artifact. Include an OPTIONAL, default-disabled nightly `parity` job that runs the same suite against a real AWS sandbox to catch local/real drift.
7. `CAPABILITY.md` — the honest emulated-vs-not table from the Verify step, kept in the repo.
8. `COST.md` — what this saves: $0 for the LocalStack path vs. real sandbox costs avoided.

## Verification & Acceptance (emit this — prove it works)
End with an "Acceptance" section the user can run:
- `docker compose up -d` then `aws --endpoint-url=http://localhost:4566 dynamodb list-tables` returns the bootstrapped table.
- `pytest tests/integration` passes locally with NO AWS credentials and NO network calls to *.amazonaws.com.
- The CI job log shows the SAME test names passing against the LocalStack service container.
- State the bar: a new developer goes from clone to a passing local integration suite in under 10 minutes, at $0 AWS spend, and the CAPABILITY table tells them exactly which behaviors still require a real-AWS check.

## Output Rules
Use only real, current SAM CLI flags, LocalStack env vars, and AWS service names. If you are not certain a flag or service-fidelity claim is correct, verify via MCP/docs or write the safe generic form — never invent one. Output each artifact as a labeled fenced code block with its file path, no preamble.
```

## What You Get

- `docker-compose.yml` — LocalStack pinned to a specific image tag, edge port 4566 exposed, an explicit `SERVICES` list, and a commented `ENFORCE_IAM=1` toggle.
- `env.json` — the SAM `--env-vars` file that injects `AWS_ENDPOINT_URL`, region, and dummy credentials so Lambda containers hit LocalStack instead of real AWS.
- A `Makefile`/`taskfile` with `up`, `down`, `bootstrap`, `invoke`, and `test` targets.
- A bootstrap script that creates the DynamoDB table (`PAY_PER_REQUEST`), S3 bucket, and SQS queue via `--endpoint-url` (or `awslocal`).
- An integration test suite (pytest+boto3 or jest+SDK v3) that round-trips DynamoDB, S3, SQS, and a Lambda invoke against the endpoint env var.
- `.github/workflows/ci.yml` — runs LocalStack as a pinned service container and executes the IDENTICAL suite, plus an optional default-off nightly real-AWS parity job.
- `CAPABILITY.md` — the honest faithfully-emulated-vs-not table.
- `COST.md` — the $0-local vs. sandbox-cost-avoided breakdown.

## Example Output

(excerpt) Reality check complete. Docker 24.0.7 (Compose v2) OK; SAM CLI 1.x OK; LocalStack image pinned to localstack/localstack:3.x. Capability table — S3: faithfully emulated (Community); DynamoDB: faithfully emulated (Community), but verify on-demand throttling on real AWS; SQS/SNS: emulated (Community); Lambda: emulated (Community), but cold-start timing and reserved concurrency are NOT representative; IAM: present but enforcement is OFF unless ENFORCE_IAM=1, so denies will NOT fire by default — verify least-privilege on real AWS; RDS / Cognito: Pro-only, not available here — STOP if your tests depend on them. Generated docker-compose.yml (4566, pinned tag), env.json (AWS_ENDPOINT_URL=http://localstack:4566), bootstrap.sh, tests/integration/test_pipeline.py, and .github/workflows/ci.yml running the same suite. Acceptance: clone-to-green in under 10 minutes at $0.

## AWS Services Used

AWS Lambda, Amazon DynamoDB, Amazon S3, Amazon SQS, Amazon SNS, Amazon API Gateway, AWS Identity and Access Management (IAM), AWS CloudFormation (via the AWS SAM transform), and the AWS SAM CLI. Tooling: LocalStack (Community edition), Docker / Docker Compose, GitHub Actions, AWS CLI v2, boto3 or AWS SDK for JavaScript v3.

## Well-Architected Alignment

- Cost Optimization: the entire inner dev loop and PR test suite run at $0 on LocalStack, eliminating per-developer sandbox stacks (a forgotten NAT Gateway alone is ~$32.40/mo); the optional real-AWS parity job runs nightly, not per-PR, to keep paid usage minimal.
- Operational Excellence: one integration suite runs identically on a laptop and in CI; pinned image tags make runs reproducible; the dev loop drops from minutes-per-`sam deploy` to seconds.
- Reliability: CI parity catches integration regressions before merge; the honest `CAPABILITY.md` prevents false confidence by naming exactly which behaviors (IAM enforcement, eventual consistency, quotas) must be confirmed on real AWS.
- Security: tests can opt into `ENFORCE_IAM=1` to validate least-privilege policies locally, with a clear instruction to confirm denies on real AWS where enforcement semantics differ.

## Cost Notes

The LocalStack Community path is $0: no AWS account, no per-API charges, no idle resources. The savings is in what it replaces — personal sandbox stacks routinely leak money: a NAT Gateway ~$32.40/mo, an idle `db.t3.micro` RDS ~$12/mo, a single Kinesis shard ~$10.80/mo. Across five developers each running a scratch stack, that is easily $200-400/month avoided. If you adopt the optional real-AWS parity job, scope it to a nightly run in a budget-capped sandbox account (recommend an AWS Budget at 80%/100% of $50/mo) so drift checking stays cheap. LocalStack Pro is a paid subscription only needed if you must emulate RDS, Cognito, ECS/EKS, or ElastiCache locally — otherwise stay on Community.

## Troubleshooting

- Cause: `sam local invoke` reaches real AWS or fails on missing credentials. Fix: the Lambda container is not on LocalStack's Docker network and `AWS_ENDPOINT_URL` was not injected — pass `--docker-network <localstack-network>` and `--env-vars env.json`, and inside the container use the LocalStack container name (e.g. http://localstack:4566), not localhost.
- Cause: an IAM-deny test "passes" locally but the policy is actually too broad. Fix: expected — LocalStack does NOT enforce IAM by default; set `ENFORCE_IAM=1` to test denies locally, and still verify least-privilege against real AWS where enforcement semantics differ.
- Cause: tests for RDS, Cognito, ECS, or ElastiCache fail or hang on Community. Fix: these are Pro-only; either move them to a real-AWS integration job or upgrade to LocalStack Pro — do not mock them in a way that fakes a pass.
- Cause: CI is flaky because the LocalStack version changed under you. Fix: pin the image to a specific tag (never `:latest`) in both `docker-compose.yml` and the CI service container so local and CI run the same build.
- Cause: a feature works locally but breaks in AWS (eventual consistency, throttling, KMS grants, cross-account access). Fix: these are not faithfully reproduced — consult `CAPABILITY.md` and gate such paths behind the nightly real-AWS parity job before relying on them.
