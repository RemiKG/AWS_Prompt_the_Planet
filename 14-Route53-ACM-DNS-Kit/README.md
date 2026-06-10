# Route 53 + ACM Done Right: End the DNS-Validation, Wrong-Region Certificate, and Failover Nightmare

Generate production-ready Terraform that validates ACM certificates over DNS in one apply, pins CloudFront certificates to us-east-1, writes legal apex ALIAS records, and fails over on Route 53 health checks—so you ship HTTPS in under 30 minutes instead of staring at `PENDING_VALIDATION` for three days.

## The Problem

Every team hits the same DNS wall: the ACM certificate sits in `PENDING_VALIDATION` and nothing tells you what to paste where. Four failures cause nearly all the pain:

- ACM allows exactly 72 hours for DNS validation. Miss it and the request flips to `VALIDATION_TIMED_OUT`—you must start over.
- The CloudFront trap: a certificate in `us-west-2` will not attach to a CloudFront distribution—CloudFront reads certificates from `us-east-1` only. People rebuild entire distributions chasing the vague console error.
- The apex record: RFC 1034 forbids a CNAME at the zone root (it cannot coexist with the SOA/NS records). Force one in and DNS breaks—email silently stops.
- No failover: an origin dies and Route 53 keeps answering with the dead target until a human notices.

## Who This Is For

Founders and engineers putting a real domain in front of a production app—a CloudFront + S3 site, an ALB-backed API, or a multi-region service that must survive a regional outage. Beginner-friendly inputs, production-grade output.

## How to Use

1. Open Claude Code or Kiro CLI in an empty `dns-kit/` directory with the AWS Documentation MCP server enabled.
2. Paste the System Prompt below verbatim.
3. Replace every bracketed placeholder: `[APEX_DOMAIN]`, `[HOSTED_ZONE_ID]`, `[ORIGIN_TYPE]` (cloudfront | alb | s3-website), `[ENABLE_DNSSEC]`, `[ENABLE_FAILOVER]`, `[PRIMARY_REGION]`, `[SECONDARY_REGION]`.
4. Let the assistant run its verify-reality checks before it writes any resource.
5. Review the files, run `terraform init` and `terraform plan`, then `terraform apply`.

Prerequisites:
- Required Access: `route53:ChangeResourceRecordSets`/`Get*`/`List*`; `acm:RequestCertificate`/`Describe*`/`List*`; `cloudfront:*` (if CloudFront); `kms:CreateKey`/`PutKeyPolicy`, `route53:CreateKeySigningKey`/`EnableHostedZoneDNSSEC` (if DNSSEC); `elasticloadbalancing:DescribeLoadBalancers` (if ALB).
- Recommended Background: a registered domain delegated to a Route 53 public hosted zone; basic Terraform.
- Tools Required: Terraform >= 1.6, AWS provider >= 5.0, AWS CLI v2, MCP tools `aws_search_documentation`/`aws_read_documentation`.

Key Parameters: certificate Region (us-east-1 ALWAYS for CloudFront and the DNSSEC KSK), validation_method (DNS, never EMAIL), key_algorithm (RSA_2048 default / EC_prime256v1 for smaller TLS handshakes), health check request_interval (30s standard / 10s fast at +$1.00/mo), failure_threshold (3), TTLs (60s validation/failover, 300s static—alias records inherit the target's TTL).

Troubleshooting: If the certificate stays `PENDING_VALIDATION` past 30 minutes, the validation CNAME is not resolving—confirm `aws_acm_certificate_validation` applied and that the CNAME (trailing dots intact) landed in the correct hosted zone.

## System Prompt

```
# Route 53 + ACM Production DNS Kit — Architecture & Terraform Request

You are a senior AWS networking engineer. Produce production-ready Terraform (>= 1.6, AWS provider >= 5.0) that wires up DNS, TLS, and failover correctly on the FIRST apply. Be opinionated, name exact resource types, state the provider versions you assume, and refuse to emit anything you cannot verify.

## My Inputs
- Apex domain: [APEX_DOMAIN]
- Public hosted zone ID: [HOSTED_ZONE_ID]
- Origin type: [ORIGIN_TYPE]  (cloudfront | alb | s3-website)
- Enable DNSSEC: [ENABLE_DNSSEC]   Enable failover: [ENABLE_FAILOVER]
- Primary region: [PRIMARY_REGION]   Secondary region: [SECONDARY_REGION]

## Verify Reality First
1. Run `aws route53 get-hosted-zone --id [HOSTED_ZONE_ID]`. Confirm the zone is PUBLIC and its four NS records match `dig NS [APEX_DOMAIN] +short`. If they differ, STOP—no validation CNAME will ever resolve.
2. Run `aws acm list-certificates` in us-east-1 and [PRIMARY_REGION]. If an ISSUED certificate already covers [APEX_DOMAIN] and `*.[APEX_DOMAIN]`, reference it via the `aws_acm_certificate` data source—never create a duplicate.
3. Confirm current ACM validation behavior, the CloudFront certificate-region rule, and DNSSEC KMS requirements via the AWS Documentation MCP server (`aws_search_documentation`, `aws_read_documentation`); cite each doc URL. If an argument changed, follow the docs, not memory.

## Hard Rules — the failures you exist to prevent
- ACM DNS validation: `validation_method = "DNS"`, one `aws_route53_record` per `domain_validation_options` entry (`allow_overwrite = true`, TTL 60), plus an explicit `aws_acm_certificate_validation` so apply BLOCKS until ISSUED. Cover [APEX_DOMAIN] and `*.[APEX_DOMAIN]` as SANs. Never EMAIL validation.
- CloudFront us-east-1 trap: for cloudfront origins the certificate's provider block MUST be aliased to `us-east-1`—CloudFront attaches certificates from no other Region. Add a [PRIMARY_REGION] certificate only if an ALB also terminates TLS.
- Apex vs CNAME: the zone root MUST be an A/AAAA `alias` record—RFC 1034 forbids an apex CNAME. CloudFront: `zone_id = "Z2FDTNDATAQYW2"`, `evaluate_target_health = false`; ALB: the load balancer's `dns_name`/`zone_id`. Prefer alias for `www`—alias queries to AWS targets are free. Add a CAA record `0 issue "amazon.com"`.
- Failover (if enabled): two records with distinct `set_identifier` values and `failover_routing_policy` PRIMARY/SECONDARY. Tie the PRIMARY to an `aws_route53_health_check` (HTTPS, port 443, `resource_path = "/health"`, `failure_threshold = 3`, `request_interval = 30`) via `health_check_id` (alias targets: `evaluate_target_health = true`).
- DNSSEC (if enabled): the KSK KMS key MUST be asymmetric, spec `ECC_NIST_P256`, usage SIGN_VERIFY, in `us-east-1`, key policy allowing `dnssec-route53.amazonaws.com` to Sign/GetPublicKey. Create `aws_route53_key_signing_key` + `aws_route53_hosted_zone_dnssec`, OUTPUT the `ds_record` for the registrar, alarm on `DNSSECInternalFailure` and `DNSSECKeySigningKeysNeedingAction`, and warn that signing caps record TTLs at one week.

## Error Management
- Zone is PRIVATE or registrar NS mismatch: stop and print the four nameservers to delegate first—validation CNAMEs cannot resolve otherwise.
- Validation hangs past 30 minutes: print `dig +short <validation_record_name> CNAME` plus the checklist: correct zone, trailing dots, delegation live.
- CloudFront rejects the certificate ARN: it lives outside us-east-1; recreate under the aliased provider—Region cannot change in place.
- DNSSEC KMS error: re-check the `dnssec-route53.amazonaws.com` key-policy principal and the key's Region.

## Deliverables
1. `providers.tf` — default provider plus an `aws.use_east_1` alias.
2. `acm.tf`, `dns.tf`, `failover.tf` (if enabled), `dnssec.tf` (if enabled), `variables.tf`, `outputs.tf`.
3. `README.md` — apply order, registrar nameserver/DS steps, and a monthly cost estimate in USD.

## Acceptance Evidence — prove it works
End with a "## Verification" section of literal commands and expected outputs: `aws acm describe-certificate --query Certificate.Status` returning ISSUED, `dig +short [APEX_DOMAIN]` returning the alias target, `dig CAA [APEX_DOMAIN] +short`, and (if DNSSEC) `dig DS [APEX_DOMAIN] +short`. Measurable bar: one clean terraform apply ending with an ISSUED, attached certificate and a resolving apex record in under 30 minutes—zero console clicks.

Output Terraform in fenced hcl code blocks with no preamble. Stay inside the 50 free AWS-endpoint health checks, default static records to 300-second TTLs, and flag every line that adds recurring spend.
```

## What You Get

- `providers.tf` — default provider plus the `aws.use_east_1` alias.
- `acm.tf` — certificate (apex + wildcard SAN), validation CNAMEs (`allow_overwrite = true`), and `aws_acm_certificate_validation` so apply blocks until ISSUED.
- `dns.tf` — apex A/AAAA alias, `www`, CAA record locking issuance to Amazon CAs.
- `failover.tf` — PRIMARY/SECONDARY records plus `aws_route53_health_check` (optional).
- `dnssec.tf` — ECC_NIST_P256 KMS KSK in us-east-1, key-signing key, zone signing, CloudWatch alarms (optional).
- `variables.tf` / `outputs.tf` — including the `ds_record` and registrar nameservers.
- `README.md` — apply order, registrar steps, cost estimate, and a `## Verification` section of copy-paste checks.

## Example Output

```hcl
# acm.tf — DNS validation that BLOCKS until ISSUED
resource "aws_acm_certificate" "cdn" {
  provider                  = aws.use_east_1   # CloudFront reads certs ONLY from us-east-1
  domain_name               = var.apex_domain
  subject_alternative_names = ["*.${var.apex_domain}"]
  validation_method         = "DNS"
  lifecycle { create_before_destroy = true }
}

resource "aws_route53_record" "cert_validation" {
  for_each        = { for dvo in aws_acm_certificate.cdn.domain_validation_options : dvo.domain_name => dvo }
  zone_id         = var.hosted_zone_id
  name            = each.value.resource_record_name
  type            = each.value.resource_record_type
  records         = [each.value.resource_record_value]
  ttl             = 60
  allow_overwrite = true
}
```

## AWS Services Used

Amazon Route 53 (hosted zones, alias records, failover routing, health checks, DNSSEC), AWS Certificate Manager (ACM), Amazon CloudFront, Elastic Load Balancing (ALB), AWS Key Management Service (KMS), Amazon S3, Amazon CloudWatch.

## Well-Architected Alignment

- Reliability: failover tied to health checks (failure_threshold 3, 30s interval); `aws_acm_certificate_validation` guarantees ISSUED before anything depends on it.
- Security: TLS via ACM, CAA restricted to Amazon CAs, optional DNSSEC with a customer-managed ECC_NIST_P256 key, least-privilege IAM.
- Operational Excellence: fully Terraform-managed; verify-reality-first checks plus alarms on `DNSSECInternalFailure` and `DNSSECKeySigningKeysNeedingAction`.
- Cost Optimization: 50 free AWS-endpoint health checks, free alias queries, 300s TTLs, every recurring line flagged.
- Performance Efficiency: apex ALIAS resolves directly to AWS targets—no extra CNAME hop; EC_prime256v1 shrinks handshakes.

## Cost Notes

Real monthly numbers (us-east-1, 2026): public hosted zone $0.50 (first 25 zones). Standard queries $0.40 per million (first billion); alias queries to AWS resources are free. Public ACM certificates are $0.00 and auto-renew. Health checks: first 50 AWS-endpoint checks free, then $0.50/mo; non-AWS endpoints $0.75/mo; optional features (HTTPS, string matching, 10s interval) $1.00/mo each AWS, $2.00/mo external. DNSSEC adds one KMS key at $1.00/mo. A CloudFront site with failover lands under $1/mo of DNS spend.

## Troubleshooting

- Certificate stuck in PENDING_VALIDATION. Cause: the validation CNAME is not resolving—wrong zone, or the registrar still points elsewhere. Fix: confirm `aws_acm_certificate_validation` applied, check `dig +short <validation_name> CNAME`, and verify delegation to the zone's four NS records before the 72-hour timeout.
- CloudFront rejects the certificate. Cause: it lives outside us-east-1. Fix: recreate it under the `aws.use_east_1` provider—Region cannot change in place.
- "CNAME already exists" at the apex. Cause: a CNAME on the zone root. Fix: replace it with an A/AAAA `alias` record and the correct alias hosted-zone ID.
- Failover never flips. Cause: the PRIMARY record has `evaluate_target_health = false` and no `health_check_id`, or the health path returns non-2xx. Fix: wire the health check to the PRIMARY and confirm 200 over HTTPS.
- DNSSEC SERVFAIL. Cause: the DS record is missing at the registrar, or the KMS key is outside us-east-1. Fix: paste the output `ds_record` and confirm ECC_NIST_P256 in us-east-1.
