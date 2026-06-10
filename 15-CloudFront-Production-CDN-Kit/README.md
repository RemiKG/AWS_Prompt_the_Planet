# CloudFront Production CDN Kit: OAC-Locked S3 Delivery with Origin Failover and Signed URLs

Generate a production-ready CloudFront distribution—Origin Access Control, a viewer-request CloudFront Function, a two-origin failover group, and time-boxed signed URLs—so a misconfigured public bucket never becomes the breach in your next due-diligence call.

## The Problem

The default startup CDN fails in three expensive ways. First, the bucket goes public to "make it work"—one wrong `s3:GetObject` grant later, private artifacts are indexed by search engines. Second, caching on the full query string lets every `?utm_source=...` link shatter the cache: a 95% hit ratio collapses toward 40% and the origin bill triples. Third, rewrite and token logic lands in Lambda@Edge at $0.60 per million requests when a CloudFront Function at $0.10 per million—submillisecond, 2 MB memory—does the job; at 500M requests a month that is $300 versus $50. Add no origin failover, and a single origin incident becomes a full outage. This prompt encodes the correct answers an AI assistant turns into deployable infrastructure.

## Who This Is For

Founding engineers and platform teams serving a SPA, static site, downloads, or private media from S3 who need a CDN that passes a security review on day one. You don't want to rediscover OAC signing, cache-key design, and signed-URL rotation by trial and error at 2 AM.

## How to Use

1. Save the System Prompt below as `cloudfront-cdn-kit.md`—in Kiro CLI under `.kiro/steering/`; in Claude Code, paste into a fresh session or `CLAUDE.md`.
2. Replace every bracketed placeholder: `[DISTRIBUTION_NAME]`, `[PRIMARY_BUCKET]`, `[FAILOVER_BUCKET]`, `[CUSTOM_DOMAIN]`, `[ACM_CERT_ARN]` (us-east-1 only), `[CONTENT_TYPE]`.
3. State Terraform or AWS CDK and confirm the AWS Documentation MCP server is connected for managed-policy and quota verification.
4. Run the prompt; review the plan, cache-key rationale, decision table, and cost estimate, then apply.
5. Validate: direct S3 curl expects 403; CloudFront curl expects 200 with `x-cache: Hit from cloudfront` on repeat.

Prerequisites:
- Required Access: an IAM principal that can create CloudFront distributions, OACs, cache policies, functions, and key groups, plus `s3:PutBucketPolicy` and `s3:PutPublicAccessBlock`; an ACM public certificate in us-east-1 for any custom domain.
- Recommended Background: S3, DNS/Route 53, and either Terraform or AWS CDK.
- Tools Required: Kiro CLI or Claude Code with the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`); the AWS API MCP server is optional.

Key Parameters: OAC signing (signing_protocol sigv4, signing_behavior always), cache key (path + query allowlist + Gzip/Brotli flags; TTL min 0s / default 86400s / max 31536000s), PriceClass_100, signed-URL TTL (300s private-media / 3600s downloads / 604800s ceiling), failover codes (500,502,503,504,403,404), HTTP/2 + HTTP/3, TLSv1.2_2021 minimum, 90-day logs, alarms (5xxErrorRate > 1%, CacheHitRate < 80%).

Troubleshooting: If direct S3 access still returns 200 instead of 403, a public ACL lingers—expected on first run; the assistant must emit an `aws s3api put-public-access-block` remediation step.

## System Prompt

```
# CloudFront Production CDN — Infrastructure Design Request

You are a senior AWS edge/CDN engineer. Produce production-ready Infrastructure-as-Code for a CloudFront distribution serving private S3 content securely, cheaply, and with automatic failover. Use Terraform unless I say AWS CDK (TypeScript). Output only the requested artifacts, no preamble.

## Verify Reality First
- Confirm both S3 buckets exist (`aws s3api get-bucket-location` or the AWS API MCP server). If one is missing, STOP and ask—never invent a bucket name.
- Confirm [ACM_CERT_ARN] exists in us-east-1; CloudFront only accepts certificates there. Otherwise STOP.
- Verify the current ID of any managed policy you reference (Managed-CachingOptimized, Managed-SecurityHeadersPolicy) via aws_read_documentation. Never hardcode an unverified ID.
- Check CloudFront distribution and OAC service quotas.
- Report which checks passed or were skipped before writing code.

## Project Overview
Distribution: [DISTRIBUTION_NAME]. Primary origin: [PRIMARY_BUCKET]. Failover origin: [FAILOVER_BUCKET]. Custom domain: [CUSTOM_DOMAIN] on certificate [ACM_CERT_ARN]. Content type: [CONTENT_TYPE] (static-site | spa | private-media | downloads).

## Detailed Requirements

### 1. Origin Security (non-negotiable)
- Origin Access Control with signing_protocol sigv4, signing_behavior always (CDK: cloudfront.Signing.SIGV4_ALWAYS). Never the legacy Origin Access Identity.
- Block Public Access stays fully ON. Bucket policies grant s3:GetObject only to cloudfront.amazonaws.com, conditioned on AWS:SourceArn matching this distribution's ARN; direct S3 GET MUST return 403.

### 2. Cache-Policy Design (justify every field in comments)
- Custom cache policy. Cache key = URL path + an explicit allowlist of query strings the origin actually varies on + enable_accept_encoding_gzip/brotli. Exclude cookies and all other headers; allowlisting keeps utm_*/fbclid from fragmenting the cache.
- TTLs: min 0s, default 86400s, max 31536000s. For spa: index.html at default TTL 0 so deploys revalidate; content-hashed assets at max TTL.

### 3. Edge Compute — Functions vs Lambda@Edge Decision Table
- Emit a decision table: price ($0.10/M, 2M free monthly vs $0.60/M plus $0.00005001 per GB-second), latency (submillisecond vs milliseconds), memory (2 MB vs configurable), package size (10 KB vs megabytes), runtime (cloudfront-js-2.0 vs Node.js/Python), capability (no network/filesystem/request-body access vs full access).
- Default to a CloudFront Function on viewer-request for URL rewrites, SPA fallback, and HMAC/JWT checks, with config in CloudFront KeyValueStore. Use Lambda@Edge ONLY when request-body access, network calls, or the AWS SDK is required—state the choice in one sentence.

### 4. Signed URLs / TTL Strategy (private-media | downloads)
- Signed URLs via a trusted key group—never legacy trusted signers tied to root-account key pairs. Expiry: 300s private-media, 3600s downloads, 604800s ceiling.
- Node.js signer: getSignedUrl from @aws-sdk/cloudfront-signer, private key from AWS Secrets Manager—never inline.
- Zero-downtime rotation: add the new public key to the key group, deploy signers with the new key-pair ID, retire the old key.

### 5. Origin Failover and Reliability
- Origin group: primary [PRIMARY_BUCKET], secondary [FAILOVER_BUCKET], failover on 500, 502, 503, 504, 403, 404. Comment that failover applies only to GET, HEAD, and OPTIONS; recommend S3 Cross-Region Replication to keep the secondary in sync.
- TLS minimum TLSv1.2_2021. Enable HTTP/2 and HTTP/3. PriceClass_100 unless I ask for global.
- Response-headers policy: HSTS, X-Content-Type-Options nosniff, Referrer-Policy strict-origin-when-cross-origin.

### 6. Observability and Cost
- Standard access logs to a dedicated S3 log bucket, 90-day lifecycle expiration.
- Monitoring subscription for additional metrics, then alarms: 5xxErrorRate > 1% (5 min), CacheHitRate < 80% (15 min).
- Monthly cost estimate at 500M requests / 5 TB egress, Function vs Lambda@Edge in dollars.

## Deliverables
1. IaC: aws_cloudfront_distribution, aws_cloudfront_origin_access_control, aws_cloudfront_cache_policy, aws_cloudfront_response_headers_policy, aws_cloudfront_function, aws_cloudfront_monitoring_subscription (or CDK).
2. CloudFront Function source (cloudfront-js-2.0), commented.
3. Both SourceArn-scoped bucket policies plus the put-public-access-block command, all four flags true.
4. Signer snippet plus aws_cloudfront_public_key and aws_cloudfront_key_group when applicable, with TTLs and rotation runbook.
5. Alarms, the log-bucket lifecycle rule, the decision table, and the cache-key rationale block.
6. Acceptance & Verification: curls proving direct S3 returns 403 and [CUSTOM_DOMAIN] returns 200 with x-cache: Hit from cloudfront on repeat, plus how to confirm failover.

## Error Management
- If a managed policy ID, quota, or bucket Region cannot be verified, STOP and report—never guess.
- Never make a bucket public to "make it work"; never emit long-lived AWS keys or an inline private key.
- If the distribution ARN is unknown before first apply, note a two-step apply instead of a circular reference. Flag every placeholder you could not resolve.

The result must deploy in under 15 minutes, return 403 on every direct S3 request, and hold a cache hit ratio above 90% within 24 hours—zero public objects, zero guessed IDs.
```

## What You Get

- Complete IaC: distribution, OAC, custom cache policy, response-headers policy, CloudFront Function (cloudfront-js-2.0), monitoring subscription—Terraform or CDK.
- Two OAC `AWS:SourceArn`-scoped bucket policies with a `put-public-access-block` command, and an origin group with the failover list plus a Cross-Region Replication recommendation.
- A Node.js signer (`@aws-sdk/cloudfront-signer`) with trusted key group, TTLs, and rotation runbook.
- Two alarms (5xxErrorRate, CacheHitRate), a 90-day log lifecycle, the decision table, a cost-estimate table, and an acceptance checklist with copy-paste curls.

## Example Output

Cache-key rationale (S3 SPA):
- path: required, content varies by route.
- query allowlist [v, lang]: the origin only varies on these; utm_* and fbclid excluded so marketing links don't fragment the cache.
- Gzip/Brotli flags on; cookies and all other headers excluded.
Projected hit ratio: 96% (vs ~42% keying the full query string).

Edge-compute decision: CloudFront Function on viewer-request (SPA fallback + HMAC check)—no body, network, or SDK access needed; $0.10/M beats $0.60/M.

## AWS Services Used

Amazon CloudFront, CloudFront Functions, CloudFront Origin Access Control (OAC), CloudFront KeyValueStore, Amazon S3, AWS Certificate Manager (ACM), Amazon Route 53, Amazon CloudWatch, AWS Secrets Manager, AWS IAM, AWS Lambda@Edge (conditional), AWS WAF (optional).

## Well-Architected Alignment

- Security: OAC `AWS:SourceArn` scoping with Block Public Access ON—zero public objects; trusted key groups; keys in Secrets Manager; TLSv1.2_2021; HSTS.
- Reliability: two-origin failover group plus Cross-Region Replication; HTTP/2 and HTTP/3; alarms on 5xx and cache hit rate.
- Cost Optimization: cache-key allowlisting; Functions over Lambda@Edge (6x cheaper); PriceClass_100; 90-day log lifecycle.
- Performance Efficiency: submillisecond edge compute, Brotli/Gzip, long TTLs for hashed assets.
- Operational Excellence: access logging, alarms, acceptance checklist, documented key rotation.

## Cost Notes

At 500M requests/month and 5 TB egress (North America): transfer ~$0.085/GB ≈ $425; HTTPS requests $0.01/10,000 ≈ $500; Functions $0.10/M ≈ $50 vs Lambda@Edge $0.60/M ≈ $300 plus GB-seconds—the Function choice saves ~$250/mo. The always-free tier covers 1 TB egress, 10M HTTPS requests, and 2M Function invocations monthly. Invalidations: 1,000 paths/month free, then $0.005 per path. The allowlisted cache key (96% vs ~42% hit ratio) keeps S3 GET and origin transfer near zero.

## Troubleshooting

- Direct S3 URL returns 200. Cause: lingering public ACL/policy. Fix: `put-public-access-block`, all four flags true; apply the OAC `AWS:SourceArn`-scoped policy.
- Cache hit ratio stuck low. Cause: full query string or cookies in the cache key. Fix: apply the allowlist cache policy; verify via `x-cache`.
- Signed URLs return AccessDenied. Cause: signer key not in the trusted key group, or clock skew. Fix: attach the public key, redeploy, confirm expiry is in the future.
- 502/504 everywhere. Cause: bucket-policy SourceArn mismatch with the distribution ARN. Fix: regenerate with the exact deployed ARN.
- Failover never triggers for writes. Cause: origin groups fail over only for GET, HEAD, and OPTIONS. Fix: route writes through a separate behavior or API origin.
