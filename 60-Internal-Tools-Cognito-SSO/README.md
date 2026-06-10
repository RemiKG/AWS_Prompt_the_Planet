# Internal Tools Behind One Cognito Login: ALB OIDC Gateway for Streamlit and Gradio on Fargate

Put every internal dashboard behind a single corporate login using the Application Load Balancer's built-in OIDC authentication action and an Amazon Cognito user pool—so your team ships data tools in an afternoon instead of hand-rolling auth five times or leaving port 8501 open to the internet.

## The Problem

Every data team ends up with a pile of internal Streamlit and Gradio tools: a sales dashboard, an ETL monitor, a prompt playground, a labeling UI, a cost explorer. Each one needs authentication, and each one usually gets it wrong in one of three ways: (1) no auth at all—Shodan indexes thousands of Streamlit servers listening on 0.0.0.0:8501; (2) a password hardcoded in a YAML file that never rotates when an employee leaves; or (3) two to three engineer-days per app wiring an OAuth library into a framework that was never designed for it—two-plus weeks of auth plumbing for a five-tool stack that generates zero revenue.

The fix is to stop putting auth in the apps. An ALB HTTPS listener rule can chain two actions—`authenticate-oidc`, then `forward`—and the load balancer performs the entire OpenID Connect dance itself: redirect to the Cognito Hosted UI, exchange the authorization code at the token endpoint, set a signed session cookie, and refresh claims until the session times out. The apps receive identity in the `x-amzn-oidc-data` header and contain zero lines of auth code. One login. Five tools. Offboard an employee with a single `admin-disable-user` call.

## Who This Is For

Startups and platform teams with 2–500 employees who need internal AI/data tools gated behind SSO today—without an identity project, without per-app logins, and without paying $8–15/user/month for a third-party access proxy. You know Docker and have run `terraform apply` once; you do not need to know OAuth.

## How to Use

1. Install the AWS Documentation MCP server in Kiro CLI, Claude Code, or Cursor (`uvx awslabs.aws-documentation-mcp-server@latest`).
2. Paste the System Prompt below into your assistant. Replace the bracketed placeholders: `[DOMAIN]` (e.g. tools.example.com), `[REGION]`, and the five tool names/ports.
3. Let the assistant run its pre-flight checks (ACM certificate, Route 53 zone, ECR images). Fix anything it reports missing before it generates code.
4. Review the Terraform plan, `terraform apply`, push your images, then run the generated `verify.sh` and add users with the generated `admin-create-user` command.

**Prerequisites**

- **Required Access:** An AWS account with permissions for ECS, Elastic Load Balancing, Cognito, ECR, Route 53, ACM, Secrets Manager, VPC, EventBridge Scheduler, and IAM role creation.
- **Recommended Background:** Basic Docker; one prior Terraform apply. No OAuth/OIDC knowledge required—that is the point.
- **Tools Required:** Terraform >= 1.7 with AWS provider >= 5.0, AWS CLI v2, Docker, and the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) so the assistant verifies current listener-rule syntax instead of guessing.

**Key Parameters:** `session_timeout` (28800s = one workday; max 604800), access/ID token validity (60 min), refresh token validity (8 h), task size (0.25 vCPU / 0.5 GB), `scope` ("openid email"), `on_unauthenticated_request` (authenticate), health-check paths (`/_stcore/health` Streamlit, `/` Gradio), off-hours scale-to-zero window (20:00–07:00 weekdays + weekends).

**Troubleshooting:** If the browser bounces between your tool and Cognito until `ERR_TOO_MANY_REDIRECTS`, the app client's callback list is missing `https://<tool-host>/oauth2/idpresponse`—the ALB always uses that exact fixed path, and it must be registered verbatim for every host.

## System Prompt

```
# Internal Tools SSO Gateway — Architecture & Terraform Request

## Project Overview
Build a production-ready single-sign-on gateway for 5 internal tools (3 Streamlit, 2 Gradio) on AWS in [REGION]. One internet-facing Application Load Balancer authenticates every request against an Amazon Cognito user pool using the ALB authentication action, then routes by hostname to ECS Fargate services in private subnets. The applications contain zero auth code. IaC: Terraform >= 1.7, AWS provider >= 5.0.

## Verify Reality Before Generating — hard requirement
1. Confirm an ACM certificate for *.[DOMAIN] exists and is ISSUED in [REGION] (`aws acm list-certificates`). If absent, emit the DNS-validated certificate resource first.
2. Confirm the Route 53 public hosted zone for [DOMAIN] exists (`aws route53 list-hosted-zones-by-name`).
3. Pull the current `aws_lb_listener_rule` authenticate-oidc argument reference via the AWS Documentation MCP server (aws_read_documentation). Do not guess deprecated syntax.
4. Confirm the 5 ECR repositories and image tags exist (`aws ecr describe-images`).
If any check fails, STOP and report exactly what is missing. Never generate HCL against unverified resources.

## Detailed Requirements

### 1. Authentication (the core)
- HTTPS:443 listener with the wildcard ACM cert (authentication actions REQUIRE HTTPS). Default action: fixed-response 404. HTTP:80 redirects to 443.
- Every routing rule chains two ordered actions: authenticate-oidc, then forward. Configuration, verbatim:
  - type = "authenticate-oidc"
  - issuer = "https://cognito-idp.[REGION].amazonaws.com/<user_pool_id>"
  - authorization_endpoint = "https://<cognito_domain_prefix>.auth.[REGION].amazoncognito.com/oauth2/authorize"
  - token_endpoint = same host, "/oauth2/token"
  - user_info_endpoint = same host, "/oauth2/userInfo"
  - client_id / client_secret from the Cognito app client (generate_secret = true; read the secret from AWS Secrets Manager, never commit it)
  - scope = "openid email"
  - session_timeout = 28800  (one 8-hour workday)
  - session_cookie_name = "AWSELBAuthSessionCookie"
  - on_unauthenticated_request = "authenticate"
  (Note: authenticate-cognito with user_pool_arn/user_pool_client_id/user_pool_domain is the same-account shorthand; keep authenticate-oidc so the pool can live in a separate security account later.)
- App client: authorization-code grant only, callback URL https://<each tool host>/oauth2/idpresponse (this exact ALB path, one entry per host), sign-out URL https://<host>/loggedout, access/ID token validity 60 minutes, refresh token validity 8 hours.
- User pool: allow_admin_create_user_only = true — self-signup DISABLED; this is an internal tool and open registration is the classic breach. Password minimum 12 characters; software-token MFA OPTIONAL.

### 2. Routing — one rule per tool
Host-header rules at priorities 10–50: sales.[DOMAIN] -> tg-sales (Streamlit :8501), etl.[DOMAIN] -> tg-etl (:8501), labeling.[DOMAIN] -> tg-labeling (:8501), playground.[DOMAIN] -> tg-playground (Gradio :7860), costs.[DOMAIN] -> tg-costs (:7860). Target groups: target_type = "ip", deregistration_delay = 30, health check /_stcore/health (Streamlit) or / (Gradio), matcher 200. Route 53 alias A records for all 5 hosts -> the ALB. Document the path-rule alternative (path_pattern /sales/* with Streamlit --server.baseUrlPath) but implement host rules — Streamlit websockets break on naive path rewrites.

### 3. Compute & Network
- One ECS cluster; 5 Fargate services, 1 task each, 0.25 vCPU / 0.5 GB (512 MiB), awsvpc, private subnets across 2 AZs, assign_public_ip = false.
- Task security group: ingress ONLY from the ALB security group on 8501/7860 (SG-to-SG reference, no CIDRs). Task execution role attaches arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy.
- No NAT gateway: interface VPC endpoints for ecr.api, ecr.dkr, and logs, plus the free S3 gateway endpoint, single-AZ ENIs (reachable VPC-wide).
- EventBridge Scheduler: ecs:UpdateService desired-count 0 at 20:00 local weekdays and all weekend, back to 1 at 07:00 — roughly 60% off Fargate spend.

### 4. Logout & Session Hygiene
Generate a ~20-line /logout handler (Python, both frameworks): expire cookies AWSELBAuthSessionCookie-0 AND -1 (the ALB shards the cookie above 4 KB), then 302 to https://<cognito_domain>/logout?client_id=<id>&logout_uri=https://<host>/loggedout. Expiring the cookie alone does NOT end the Cognito Hosted UI session.

### 5. Cost Target
Under $100/month all-in for 5 tools and 50 users. An itemized monthly estimate is a required deliverable.

## Deliverables Requested
1. Terraform modules — network/, auth/, alb/, service/ (instantiated 5x) — plan-clean, no warnings.
2. One Dockerfile per framework with the logout route wired in and a non-root user.
3. verify.sh — evidence the stack works: curl -sI each host expects 302 to /oauth2/authorize; replay with a valid session cookie expects 200; /logout expects Set-Cookie expiry plus 302 to Cognito /logout. Print PASS/FAIL per check with expected output shown.
4. Per-line cost table (ALB hours + LCU, Fargate vCPU/GB, VPC endpoints, Route 53, CloudWatch Logs, Cognito tier).
5. ONBOARDING.md — the exact `aws cognito-idp admin-create-user` and `admin-disable-user` commands for the team.

Align with the Well-Architected Security and Cost Optimization pillars. Output complete files only, no preamble. The stack must deploy with a single terraform apply in under 15 minutes, and adding tool #6 must require only one module instantiation and one DNS record.
```

## What You Get

1. **Terraform, 4 modules (~14 files):** `network/` (VPC, 2 private + 2 public subnets, 4 VPC endpoints), `auth/` (Cognito user pool with self-signup disabled, hosted-UI domain, ALB app client, Secrets Manager secret), `alb/` (HTTPS listener, 5 authenticate-oidc + forward rules at priorities 10–50, Route 53 records), `service/` (target group, ECS service, task definition, scale-to-zero schedules) instantiated once per tool.
2. **Two hardened Dockerfiles** (Streamlit and Gradio) with the `/logout` route and non-root user baked in.
3. **`logout.py`** — the 20-line cookie-expiry + Cognito `/logout` redirect handler.
4. **`verify.sh`** — automated proof: unauthenticated 302, authenticated 200, working logout.
5. **Itemized cost table** and **`ONBOARDING.md`** with copy-paste user add/disable commands.

## Example Output

```
[pre-flight] ACM cert arn:aws:acm:us-east-1:...:certificate/3f1c... ISSUED for *.tools.example.com  OK
[pre-flight] Route 53 zone Z04857EXAMPLE found for tools.example.com  OK
[pre-flight] ECR: 5/5 repositories have tag v1  OK
wrote modules/alb/rules.tf  — 5 host rules, each authenticate-oidc (session_timeout=28800) -> forward
wrote modules/auth/cognito.tf — user pool "internal-tools": allow_admin_create_user_only=true

$ ./verify.sh
sales.tools.example.com        -> 302 .../oauth2/authorize?client_id=...   PASS
sales.tools.example.com +cookie -> 200 OK (412 ms)                          PASS
/logout -> Set-Cookie: AWSELBAuthSessionCookie-0=; Expires=1970 -> Cognito  PASS
Estimated monthly: $87.88 (ALB 19.43 | Fargate 45.05 | VPCe 21.90 | R53 0.50 | Logs 1.00 | Cognito 0.00)
```

## AWS Services Used

Amazon ECS on AWS Fargate, Elastic Load Balancing (Application Load Balancer), Amazon Cognito, Amazon ECR, Amazon Route 53, AWS Certificate Manager, AWS Secrets Manager, Amazon VPC (interface + gateway endpoints), Amazon EventBridge Scheduler, Amazon CloudWatch Logs, AWS IAM.

## Well-Architected Alignment

- **Security:** Tasks in private subnets with no public IPs; task SG accepts traffic only from the ALB SG; self-signup disabled (`allow_admin_create_user_only`); client secret in Secrets Manager; HTTPS-only with managed ACM cert; identity delivered via the signed `x-amzn-oidc-data` header (verify its ES256 signature against `https://public-keys.auth.elb.<region>.amazonaws.com/<kid>` for defense in depth).
- **Cost Optimization:** 0.25 vCPU tasks; scheduled scale-to-zero outside working hours; VPC endpoints instead of a NAT gateway; Cognito free for the first 10,000 MAUs.
- **Operational Excellence:** Everything in Terraform; `verify.sh` turns "it works" into evidence; onboarding/offboarding is one CLI command.
- **Reliability:** Two AZs, per-tool target-group health checks, 30-second deregistration delay for clean deploys.

## Cost Notes

Five tools, 50 users, us-east-1, 24/7:

| Line item | Math | Monthly |
|---|---|---|
| ALB hours | $0.0225 × 730 h | $16.43 |
| ALB LCU (light internal traffic) | ~0.5 LCU avg × $0.008 × 730 | ~$3.00 |
| Fargate, 5 × (0.25 vCPU/0.5 GB) | 5 × ($0.01012 + $0.00222) × 730 | $45.05 |
| VPC interface endpoints ×3 | 3 × $0.01 × 730 (single-AZ) | $21.90 |
| Route 53 hosted zone | flat | $0.50 |
| CloudWatch Logs (~2 GB) | $0.50/GB ingest | $1.00 |
| Cognito Essentials, 50 MAU | first 10,000 MAUs free | $0.00 |
| **Total** | | **~$87.88 (≈$17.60/tool)** |

With the EventBridge scale-to-zero schedule (≈282 task-hours/month instead of 730), Fargate drops to ~$17.40 and the stack lands near **$60/month**. Compare: a commercial access proxy at $8/user × 50 users = $400/month for the auth layer alone.

## Troubleshooting

1. **Redirect loop, then ERR_TOO_MANY_REDIRECTS.** Cause: callback URL `https://<host>/oauth2/idpresponse` missing from the app client for that host. Fix: add it verbatim — the ALB's redirect path is fixed and per-host.
2. **ALB returns HTTP 561.** Cause: the token-endpoint exchange failed, almost always a wrong or rotated `client_secret`. Fix: re-read the secret from Secrets Manager, update the listener rule, re-apply.
3. **Streamlit loads then hangs on "Please wait…".** Cause: path-based routing broke the websocket base path. Fix: use host-header rules, or set `--server.baseUrlPath` to match the path rule exactly.
4. **Logout button doesn't actually log out.** Cause: only one cookie shard expired, or the Cognito Hosted UI session survived. Fix: expire both `AWSELBAuthSessionCookie-0` and `-1`, redirect to Cognito `/logout` with a `logout_uri` registered in the app client's sign-out URLs.
5. **Tasks stuck in PROVISIONING / image pull fails.** Cause: private subnets with no NAT and missing endpoints. Fix: confirm ecr.api, ecr.dkr interface endpoints and the S3 gateway endpoint exist and share the task subnets' route tables/SGs.
