# PCI DSS SAQ A-EP Compliance Kit: AWS WAF, VPC Segmentation, CloudTrail, and KMS Mapped to Every Checklist Question

Generate a complete requirement-to-control mapping table for the PCI DSS v4.0.1 SAQ A-EP checklist plus the Terraform that implements every automatable control—so your e-commerce payment page passes self-assessment instead of triggering a six-figure audit scramble.

## The Problem

Your startup uses Stripe, Braintree, or Adyen, so you assume you qualify for SAQ A—about 30 questions, done in an afternoon. Then your acquirer looks at your integration: your server delivers the payment form (Direct Post or merchant-hosted JavaScript fields), which means your website *affects the security of the transaction*. You're now in SAQ A-EP territory: roughly **190 questions across all 12 PCI DSS requirement families**, with your entire web tier in scope.

The stakes are real. PCI DSS v3.2.1 retired on March 31, 2024; v4.0.1's future-dated requirements—including mandatory automated web-attack protection (Req 6.4.2), payment-page script management (Req 6.4.3), and weekly payment-page tamper detection (Req 11.6.1)—became enforceable on **March 31, 2025**. Card brands can assess non-compliance fees of $5,000–$100,000 per month against your acquirer, which passes them straight to you. A QSA-led gap assessment runs $15,000–$40,000—and most of what they'd tell you is mechanical: which AWS control answers which checklist line.

This prompt does that mechanical work. It verifies what actually exists in your account, maps every SAQ A-EP requirement to a concrete AWS control (or flags it as Manual with an owner), and generates production-ready Terraform for segmentation, AWS WAF, CloudTrail, and KMS—the four control planes that cover the bulk of the technical questions.

## Who This Is For

- E-commerce startups whose checkout page hosts the processor's JavaScript or posts card data directly to the processor (the classic SAQ A-EP trigger)
- CTOs/founding engineers handling their first PCI self-assessment without a compliance hire
- Platform engineers who already passed SOC 2 or HIPAA and discover PCI demands segmentation evidence neither framework required
- Teams whose acquirer just bumped them from SAQ A to SAQ A-EP after a v4.0.1 eligibility review

## How to Use

1. Save the System Prompt below as `pci-saq-aep.md` in your infrastructure repo (Kiro users: place it in `.kiro/steering/`).
2. Open Claude Code, Kiro CLI, or Amazon Q Developer CLI in that repo with AWS credentials for the target account loaded (read-only is enough for the verification pass).
3. Replace the bracketed placeholders: `[REGION]`, `[PAYMENT_PROCESSOR]`, `[CHECKOUT_DOMAIN]`, `[CDN: CloudFront or ALB-only]`.
4. Paste the prompt. Let the assistant run its read-only discovery commands **before** it generates anything—if it skips verification, stop it and re-paste.
5. Review `saq-aep-control-map.md` and `gap-register.md` first; the mapping table is the artifact your acquirer or QSA will actually ask for.
6. Run `terraform init && terraform plan`, review, then `apply`. Execute the acceptance commands and file the outputs as evidence.

**Prerequisites**

- **Required Access:** AWS account with admin (or PowerUserAccess + IAM) for apply; `SecurityAudit` managed policy is sufficient for the discovery pass. Terraform v1.7+, AWS provider v5.x.
- **Recommended Background:** Basic Terraform; a confirmed SAQ type from your acquirer (this kit targets SAQ A-EP; if you're full-redirect/iframe you may only need SAQ A).
- **Tools Required:** AWS CLI v2; AWS Documentation MCP server (`aws_search_documentation`, `aws_read_documentation`) so the assistant verifies control names and regional availability against live docs instead of memory.

**Key Parameters:** WAF rate limit (2,000 req/5 min), CloudTrail retention (400d S3 Object Lock / 90d CloudWatch Logs hot per Req 10.5.1), KMS rotation (annual, 4 CMKs), TLS floor (TLSv1.2_2021), IAM password minimum (12 chars per 8.3.6), inactive-account disable (90d per 8.2.6), payment-page integrity cadence (7d per 11.6.1), monthly cost ceiling ($75).

**Troubleshooting:** If the assistant creates the WAF web ACL in your home region and CloudFront association fails, that's expected—web ACLs with `CLOUDFRONT` scope can only be created in us-east-1. Re-run the WAF module with a us-east-1 provider alias.

## System Prompt

```
# PCI DSS v4.0.1 SAQ A-EP Control Implementation Request

## Project Overview
I run an e-commerce site on AWS in [REGION] that qualifies for PCI DSS v4.0.1 SAQ A-EP: all cardholder data is captured, processed, and stored by [PAYMENT_PROCESSOR] (a PCI-validated third party), but my web server at [CHECKOUT_DOMAIN] delivers the payment page ([CDN: CloudFront or ALB-only] in front), so my web tier is in scope. Design the production-ready control set, generate the Terraform (v1.7+, AWS provider v5.x preferred), and map every control back to the SAQ A-EP checklist.

## Verify Reality First — before generating anything
1. Run `aws sts get-caller-identity` and `aws ec2 describe-vpcs --region [REGION]` to confirm the account and whether an in-scope web tier already exists. Never invent VPC, subnet, or distribution IDs; discover them or declare them as new resources.
2. Use the AWS Documentation MCP server (aws_read_documentation) to confirm AWS WAF managed rule group names and Security Hub PCI DSS v4.0.1 availability in [REGION]. Confirm the exact standard ARN with `aws securityhub describe-standards` before enabling it.
3. If the CDN is CloudFront, the web ACL MUST use scope CLOUDFRONT and be created in us-east-1.
4. If any check fails or a service is unavailable in [REGION], stop and report the gap. Do not generate plausible-looking configuration for resources you could not verify.

## Detailed Requirements
### 1. Segmentation (Req 1)
- Dedicated VPC for the in-scope web tier: public /24 subnets in 2 AZs, security groups default-deny, inbound 443 only (from the CloudFront origin-facing managed prefix list if CloudFront is used), zero inbound port 22 — administration via SSM Session Manager only
- VPC Flow Logs to CloudWatch Logs, 90-day retention
### 2. Public-Facing Web Application Protection (Req 6.4.2)
- One AWS WAF web ACL: AWSManagedRulesCommonRuleSet, AWSManagedRulesKnownBadInputsRuleSet, AWSManagedRulesSQLiRuleSet, AWSManagedRulesAmazonIpReputationList, plus a rate-based rule at 2,000 requests/5 minutes; CloudWatch metrics and sampled requests enabled
### 3. Logging & Monitoring (Req 10, 11.5)
- CloudTrail with log file validation enabled, delivered to a dedicated S3 bucket with Object Lock in compliance mode (400-day retention) and mirrored to CloudWatch Logs (90-day retention) — satisfying Req 10.5.1 (12 months retained, 3 months immediately available)
- GuardDuty enabled as the intrusion-detection technique (Req 11.5.1); Security Hub with the PCI DSS v4.0.1 standard (144 automated controls); SNS email alert on CRITICAL findings
### 4. Encryption & Key Management (Req 3, 4)
- Four customer-managed KMS keys (CloudTrail, S3 logs, EBS, Secrets Manager) with annual automatic rotation; CloudTrail key policy must grant cloudtrail.amazonaws.com with an EncryptionContext condition
- TLS 1.2 minimum end to end: CloudFront security policy TLSv1.2_2021, ALB policy ELBSecurityPolicy-TLS13-1-2-2021-06
### 5. Identity (Req 7, 8)
- IAM password policy 12-character minimum (8.3.6), MFA enforced (8.4.2), credentials disabled after 90 days of inactivity (8.2.6), least-privilege roles instead of shared users (7.2)
### 6. Payment Page Integrity (Req 6.4.3, 11.6.1)
- Inventory and authorize every payment-page script; emit Content-Security-Policy and Subresource Integrity via a CloudFront response headers policy; schedule a tamper-detection check of the payment page and its HTTP headers at least every 7 days
### 7. Cost Ceiling
- The full control set must run under $75/month for a single account at 5M requests/month; flag any line item that exceeds it

## Deliverables Requested
1. `saq-aep-control-map.md` — the requirement-to-control mapping table covering every SAQ A-EP requirement family. Columns: SAQ A-EP requirement number | requirement summary | AWS control implemented | Terraform resource address | evidence artifact and where it is stored | status (Automated / Manual / Shared-with-AWS / N-A). Mark requirements AWS cannot satisfy — quarterly external ASV scans (11.3.2), security policies and TPSP management (Req 12) — as Manual with a named owner placeholder. Mark physical security (Req 9) as Shared-with-AWS, evidenced by the AWS PCI DSS AOC from AWS Artifact.
2. Terraform: `network.tf`, `waf.tf`, `logging.tf`, `kms.tf`, `iam.tf`, `monitoring.tf`, plus `variables.tf` with the defaults above as tunable variables.
3. `evidence/collection.md` — one exact AWS CLI command per automated control that pulls assessment evidence.
4. `gap-register.md` — every Manual item with owner, effort estimate, and due date column.
5. A per-service monthly cost table.

## Acceptance Evidence
End with a verification section listing the exact commands and expected values proving the deployed controls work: `aws cloudtrail get-trail-status` (IsLogging: true), `aws cloudtrail describe-trails` (LogFileValidationEnabled: true), `aws kms get-key-rotation-status` (KeyRotationEnabled: true), `aws wafv2 get-web-acl --scope CLOUDFRONT --region us-east-1`, `aws securityhub get-enabled-standards`. State plainly that SAQ A-EP is a self-assessment: this kit produces controls and evidence, not a certification, and the ASV scan and policy documents remain the merchant's responsibility.

Align with the AWS Well-Architected Security and Cost Optimization pillars. The Terraform must be deployable with a single `terraform apply` in under 15 minutes. Output the mapping table first, then the Terraform, without any preamble.
```

## What You Get

1. **`saq-aep-control-map.md`** — the requirement-to-control mapping table for the full SAQ A-EP checklist: every requirement line mapped to an AWS control, Terraform resource address, evidence artifact, and an Automated / Manual / Shared-with-AWS / N-A status. This is the document you hand your acquirer.
2. **Seven Terraform files** — `network.tf` (segmented VPC, security groups, Flow Logs), `waf.tf` (web ACL + 4 managed rule groups + rate rule), `logging.tf` (CloudTrail, Object Lock bucket, CloudWatch Logs), `kms.tf` (4 CMKs with rotation), `iam.tf` (password policy, MFA enforcement), `monitoring.tf` (GuardDuty, Security Hub PCI v4.0.1, SNS alerts), `variables.tf`.
3. **`evidence/collection.md`** — copy-paste CLI commands that pull dated evidence for each automated control.
4. **`gap-register.md`** — every Manual item (ASV scans, policies, TPSP list per 12.8) with owner and due-date columns.
5. **Monthly cost table** — per-service, checked against the $75 ceiling.
6. **Acceptance section** — the exact commands and expected outputs proving each control is live.

## Example Output

| SAQ A-EP Req | Summary | AWS Control | Terraform | Evidence | Status |
|---|---|---|---|---|---|
| 1.3.1 | Inbound traffic to CDE restricted | SG allows 443 only from CloudFront origin-facing prefix list | `aws_security_group.web` | `describe-security-groups` → evidence/req-1/ | Automated |
| 6.4.2 | Automated attack detection for public web app | WAF web ACL: 4 AWS managed rule groups + 2,000 req/5 min rate rule | `aws_wafv2_web_acl.checkout` | `get-web-acl` → evidence/req-6/ | Automated |
| 10.5.1 | 12 months audit log retention, 3 months online | CloudTrail → S3 Object Lock 400d + CloudWatch Logs 90d | `aws_cloudtrail.main` | `get-trail-status` → evidence/req-10/ | Automated |
| 11.3.2 | Quarterly external ASV scan | Not satisfiable by AWS tooling — Inspector is not an ASV | — | ASV report upload | Manual (owner: TBD) |
| 9.x | Physical access to CDE | AWS data-center controls | — | AWS PCI AOC via AWS Artifact | Shared-with-AWS |

## AWS Services Used

AWS WAF, Amazon VPC, AWS CloudTrail, AWS KMS, AWS Security Hub, Amazon GuardDuty, Amazon CloudFront, AWS IAM, Amazon S3, Amazon CloudWatch, AWS Systems Manager (Session Manager), Amazon SNS, Amazon Inspector, AWS Secrets Manager, AWS Artifact

## Well-Architected Alignment

- **Security:** Defense in depth across all seven layers PCI cares about—edge (WAF managed rule groups), network (default-deny security groups, no port 22), identity (MFA, 12-char minimum, 90-day inactivity disable), data (KMS CMKs with annual rotation, TLS 1.2 floor), and detection (GuardDuty, Security Hub PCI v4.0.1 with 144 automated controls).
- **Operational Excellence:** Evidence collection is scripted, not tribal knowledge; the gap register turns compliance into a tracked backlog; acceptance commands make "is it on?" answerable in one terminal session.
- **Cost Optimization:** A hard $75/month ceiling baked into the prompt; one web ACL shared across the checkout path; CloudWatch hot retention capped at the 90 days Req 10.5.1 actually demands, with cheap S3 Object Lock carrying the remaining 12 months.
- **Reliability:** Two-AZ subnet layout, immutable compliance-mode logs that survive accidental deletion, and SNS alerting on CRITICAL findings.

## Cost Notes

For a single account serving 5M checkout-path requests/month:

- **AWS WAF:** $5.00 web ACL + $1.00 × 4 managed rule groups + $1.00 rate rule + $0.60/M requests (~$3.00) ≈ **$13/month**
- **AWS KMS:** 4 customer-managed keys × $1.00 + API calls at $0.03/10K ≈ **$4–5/month**
- **CloudTrail:** first copy of management events is free; S3 storage at $0.023/GB-month plus CloudWatch Logs ingestion at $0.50/GB ≈ **$3–8/month**
- **GuardDuty:** ≈ **$5–15/month** at small-account event volume
- **Security Hub:** ≈ **$10–20/month** with PCI v4.0.1 enabled in one account/region
- **Amazon Inspector:** ≈ **$1.25 per EC2 instance/month**

Total ≈ **$40–65/month**—inside the $75 ceiling, versus $15,000–$40,000 for a QSA gap assessment. Budget separately for the mandatory quarterly ASV scans (typically a few hundred to ~$2,000/year from an Approved Scanning Vendor); no AWS pre-approval is required for scanning permitted services.

## Troubleshooting

1. **WAF web ACL won't associate with CloudFront.** Cause: web ACLs with `CLOUDFRONT` scope only exist in us-east-1. Fix: add a us-east-1 provider alias and recreate the web ACL there; regional ALB-only setups use `REGIONAL` scope in the home region.
2. **Can't enable Object Lock on the existing CloudTrail bucket.** Cause: S3 Object Lock can only be enabled at bucket creation. Fix: create a new bucket with `--object-lock-enabled-for-bucket`, point the trail at it, and keep the old bucket until its retention lapses.
3. **CloudTrail stops delivering after KMS encryption is added.** Cause: the key policy doesn't grant `cloudtrail.amazonaws.com`. Fix: add a key-policy statement allowing `kms:GenerateDataKey*` with an `EncryptionContext` condition scoped to your trail ARN.
4. **Security Hub PCI v4.0.1 fails controls on out-of-scope resources.** Cause: the standard evaluates the whole account, not just the CDE. Fix: move out-of-scope workloads to a separate account (the cleanest segmentation evidence), or disable specific controls with a written justification recorded in `gap-register.md`.
5. **The mapping table marks 11.3.2 as Manual and you expected automation.** Cause: Amazon Inspector is not a PCI SSC Approved Scanning Vendor; external quarterly scans must come from an ASV. Fix: contract an ASV, schedule quarterly scans against `[CHECKOUT_DOMAIN]`, and file the passing report as evidence.
