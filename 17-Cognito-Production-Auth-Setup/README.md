# Cognito Production Auth From Scratch: Hosted UI Custom Domain, OAuth, and MFA Without Broken Redirect Flows

Stand up a production-ready Amazon Cognito user pool with custom-domain hosted UI, PKCE authorization-code flow, federation, and enforced MFA—so you stop debugging `redirect_mismatch` and start shipping authenticated features.

## The Problem

Cognito is the fastest way to add auth on AWS, and the fastest way to lose a weekend. The hosted UI custom domain demands an ACM certificate in `us-east-1` only (it sits behind Cognito-managed CloudFront) plus a DNS **A record on the parent domain**—miss either and `CreateUserPoolDomain` fails with an error it never explains. The authorization-code flow breaks silently: a client secret a SPA cannot send, a callback URL off by one trailing slash, an unchecked code grant, or Terraform's `allowed_oauth_flows_user_pool_client` left `false` so every OAuth setting is ignored. Default token lifetimes (ID/access 60 minutes, refresh 30 days) leave a stolen refresh token valid for a month; blind tuning logs users out. Teams burn 6-10 hours here, then ship MFA as OPTIONAL. This prompt generates the whole stack correctly the first time, with the exact app-client settings that break OAuth called out before they bite.

## Who This Is For

- Founders and full-stack engineers adding login to a SaaS app or internal tool
- Teams replacing a hand-rolled auth service with managed Cognito
- Anyone stuck on `redirect_mismatch`, `invalid_grant`, or a custom-domain failure who wants the correct configuration, not a forum thread

## How to Use

1. Open your AI assistant (Kiro CLI, Claude Code, Amazon Q Developer, or Cursor) in your auth/IaC repo.
2. Connect the AWS Documentation MCP server for current Cognito API shapes and feature plans.
3. Paste the full prompt from the System Prompt section below.
4. Replace every bracketed placeholder: `[APP_DOMAIN]`, `[AUTH_SUBDOMAIN]`, `[CALLBACK_URL]`, `[LOGOUT_URL]`, `[AWS_REGION]`, `[IDP]`.
5. Let the assistant verify reality first, then generate the IaC, DNS records, and verification runbook.
6. Deploy, then run the printed acceptance commands: hosted UI loads, code grant returns tokens, MFA enforced.

Prerequisites:
- Required Access: `cognito-idp:*` on the target pool, `acm:RequestCertificate`/`acm:DescribeCertificate`, `route53:ChangeResourceRecordSets`, and `cloudfront:UpdateDistribution` (needed to bind the custom domain).
- Recommended Background: OAuth 2.0 basics, DNS records, one IaC tool.
- Tools Required: AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`), AWS CLI v2, Terraform >= 1.5 or AWS CDK v2.

Key Parameters: ACM cert region (us-east-1 ONLY), auth subdomain (auth.[APP_DOMAIN]), feature plan (ESSENTIALS default; PLUS for threat protection), token lifetimes (access/ID 15min, refresh 24h-7d vs defaults 60min/30d), OAuth (code grant ON, implicit OFF, `allowed_oauth_flows_user_pool_client` true), scopes (openid/email/profile), MFA REQUIRED with TOTP primary, prevent-user-existence-errors ENABLED, token revocation + refresh rotation ON.

Troubleshooting: If `CreateUserPoolDomain` fails with a parent-domain message, it is expected—Cognito requires a resolvable DNS A record on the PARENT of your auth subdomain before it provisions the CloudFront distribution. Create that record first, then retry.

## System Prompt

```
# Cognito Production Auth Setup — Architecture & IaC Request

You are a senior AWS identity engineer. Configure a PRODUCTION-READY Amazon Cognito user pool with a custom-domain hosted UI, OAuth 2.0 authorization-code flow with PKCE, federation, and enforced MFA. Output IaC plus an exact DNS and verification runbook. Every setting below exists because the default silently breaks a production login flow.

## Verify Reality FIRST
1. Use the AWS Documentation MCP tools (aws_read_documentation, aws_search_documentation) to confirm current CreateUserPoolDomain, CreateUserPoolClient, and SetUserPoolMfaConfig shapes, token-lifetime bounds, and feature plans. Do not rely on memory.
2. Confirm the pool region is [AWS_REGION]. State that the ACM certificate MUST live in us-east-1 regardless—Cognito fronts the hosted UI with a global CloudFront distribution.
3. Run read-only discovery: aws cognito-idp list-user-pools --max-results 20, aws acm list-certificates --region us-east-1, a DNS lookup on [APP_DOMAIN]. If any exist, READ and reconfigure—never recreate; flag anything that would delete users as a hard stop.
4. If you cannot confirm a value, say so and emit the safe generic form. Inventing an API, flag, or quota is a hard stop.

## Project Inputs
[APP_DOMAIN], [AUTH_SUBDOMAIN] (recommend auth.[APP_DOMAIN]), [CALLBACK_URL], [LOGOUT_URL], [AWS_REGION], [IDP] (e.g. Google, SAML).

## Detailed Requirements

### 1. User Pool
- Email sign-in, case-insensitive aliases. Password policy: 12+ chars, upper/lower/number/symbol.
- prevent_user_existence_errors = ENABLED; deletion_protection = ACTIVE.
- Feature plan: Essentials (default) suffices; threat protection needs Plus and user_pool_add_ons { advanced_security_mode = "ENFORCED" }.

### 2. Hosted UI Custom Domain (the part that breaks)
- DNS-validated ACM cert in us-east-1 for [AUTH_SUBDOMAIN]: aws_acm_certificate + aws_acm_certificate_validation under a provider alias.
- BEFORE CreateUserPoolDomain: a resolvable A record MUST exist on the PARENT domain ([APP_DOMAIN]); Cognito refuses to provision without it. Emit as an ordered step.
- Create aws_cognito_user_pool_domain, capture the CloudFront alias target, emit the Route 53 alias A record for [AUTH_SUBDOMAIN]. Distribution takes up to ~1 hour; the runbook must wait-and-verify.

### 3. App Client + OAuth (the exact settings that break flows)
- Browser/SPA = PUBLIC client, generate_secret = false (a browser cannot keep a secret); server-rendered = confidential WITH a secret. Say which and why.
- allowed_oauth_flows = ["code"] only; implicit OFF (leaks tokens in URLs).
- allowed_oauth_flows_user_pool_client = true. If false, all OAuth settings are silently ignored and /oauth2/authorize returns unauthorized_client.
- allowed_oauth_scopes = openid, email, profile; custom scopes only with a resource server.
- callback_urls/logout_urls match requests byte-for-byte (trailing slash, scheme, port); HTTPS except http://localhost. Enumerate exact strings.
- PKCE, enable_token_revocation = true, refresh-token rotation ON with retry grace period.

### 4. Token Lifetimes (each with its reason)
- Access 15 min (limits stolen-token blast radius); ID 15 min (refreshes alongside access); refresh 24h-7d with rotation ON (the 30-day default keeps a stolen refresh token valid a month). State chosen values AND reasons. Bounds: access/ID 5 min-24 h, refresh 60 min-10 years.

### 5. MFA + Federation
- mfa_configuration = ON, not OPTIONAL. TOTP primary; SMS fallback only if demanded (SNS spend limit, originating identity, sms_configuration role, per-message cost).
- Per provider in [IDP]: aws_cognito_identity_provider mapping email + sub, added to supported_identity_providers.

## Error Management
The runbook must list Cause -> Resolution for: redirect_mismatch/invalid_grant (callback byte mismatch or code reuse), unauthorized_client (code grant unchecked, scope missing, or allowed_oauth_flows_user_pool_client false), SPA exchange failure (client secret present), parent-domain error (missing parent A record), certificate errors (cert not in us-east-1 or CloudFront still distributing, up to 1 hour).

## Deliverables (every file)
1. cognito.tf — pool, client(s), MFA, feature-plan add-ons. 2. acm.tf — us-east-1 alias + DNS-validated cert. 3. dns.tf — parent A record + [AUTH_SUBDOMAIN] alias. 4. idp.tf — providers + mapping. 5. runbook.md — exact hosted-UI test URL + curl /oauth2/token exchange. 6. cost-notes.md — MAU by feature plan + SMS.

## Evidence & Acceptance
End with a ## Verification section: commands proving HTTP 200 on https://[AUTH_SUBDOMAIN]/login, id/access/refresh tokens from the code grant, 401 from unauthenticated /oauth2/userInfo, and an MFA challenge after password—expected output pasted under each. Configuration without proof is not done.

## Output Rules
Apply-clean IaC, pinned provider versions, no console steps beyond the DNS-validation wait. No preamble. Target: terraform apply to verified login in under 30 minutes.
```

## What You Get

- Terraform (or CDK v2) split as `cognito.tf`, `acm.tf` (us-east-1 provider + DNS-validated cert), `dns.tf` (parent A record + auth alias to CloudFront), `idp.tf`, plus `runbook.md` (sign-in URL, `curl` code exchange, Cause -> Resolution for the five failure modes) and `cost-notes.md` (MAU by feature plan + SMS).
- A `## Verification` section — proof of HTTP 200 on the hosted UI, tokens from the code grant, 401 on unauthenticated `/oauth2/userInfo`, enforced MFA.

## Example Output

Step 4 — Test the hosted UI custom domain:
https://auth.example.com/login?client_id=2a1b3c4d5e6f&response_type=code&scope=openid+email+profile&redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback
Expected: managed login page loads (HTTP 200), MFA challenge after password.

Step 5 — Exchange the code (public SPA client, PKCE, no secret):
curl -X POST https://auth.example.com/oauth2/token -H "Content-Type: application/x-www-form-urlencoded" -d "grant_type=authorization_code&client_id=2a1b3c4d5e6f&code=AUTH_CODE&redirect_uri=https://app.example.com/callback&code_verifier=VERIFIER"
Expected: 200 with id_token, access_token, refresh_token. A 400 invalid_grant means redirect_uri mismatch or code reuse.

## AWS Services Used

Amazon Cognito (user pools, hosted UI / managed login, OAuth 2.0, MFA), AWS Certificate Manager (ACM), Amazon CloudFront (Cognito-managed custom domain), Amazon Route 53, Amazon SNS (SMS MFA), AWS IAM, Terraform or AWS CDK.

## Well-Architected Alignment

- Security: enforced MFA, threat protection (ENFORCED, Plus plan), PKCE, prevent-user-existence-errors, 15-minute access tokens, refresh-token rotation, least-privilege deploy IAM.
- Operational Excellence: fully IaC-driven, with a runnable verification section and deterministic DNS/cert ordering—no undocumented console clicks.
- Reliability: deletion protection, auto-renewing ACM certificate, explicit handling of the up-to-1-hour domain propagation.
- Cost Optimization: per-feature-plan MAU estimate, TOTP-first MFA, an SNS spend limit.
- Performance Efficiency: token caching (reuse to ~75% of lifetime) to stay under Cognito token-endpoint quotas.

## Cost Notes

Cognito bills per monthly active user (MAU) under feature plans. Essentials—the default for new pools and sufficient here—is $0.015 per MAU beyond a 10,000-MAU free tier (~$150/month for 10,000 paid MAUs). Lite keeps legacy $0.0055-per-MAU tiering; Plus ($0.02 per MAU) is required for threat protection (adaptive auth, compromised-credential detection). SAML/OIDC-federated users get a separate 50-MAU free tier. SMS MFA bills through Amazon SNS at destination-dependent rates—prefer TOTP to keep this near zero. ACM public certificates are free, Cognito's CloudFront distribution adds no separate charge, and Route 53 hosted zones are $0.50/month.

## Troubleshooting

- redirect_mismatch / invalid_grant: the callback URL differs from the request byte-for-byte (trailing slash, http vs https, port). Fix: make them identical; HTTPS except http://localhost.
- unauthorized_client: code grant off, scope not allowed, or `allowed_oauth_flows_user_pool_client` false (OAuth config silently ignored). Fix: enable the code grant, allow openid/email/profile, set the flag true.
- SPA token exchange fails: the client has a secret a browser cannot send. Fix: public client, no secret, PKCE.
- CreateUserPoolDomain parent-domain error: no A record on the parent domain. Fix: create one (any resolvable value), retry.
- Certificate errors on the new domain: cert not in us-east-1, or CloudFront still distributing. Fix: issue in us-east-1 only; allow ~1 hour.
