# VPC Lattice Service Network Kit: IAM-Authenticated Cross-Account Microservices Without Sidecars or CIDR Surgery

Application-layer service-to-service connectivity across VPCs and accounts with deny-by-default IAM auth policies—ship cross-team APIs in an afternoon instead of running a service mesh or re-IPing half your estate.

## The Problem

Your `orders` service in account A needs to call `payments` in account B. The classic answers all hurt: Transit Gateway + internal ALBs costs $0.05/attachment-hour per VPC plus $0.02/GB and dies instantly when two VPCs overlap on 10.0.0.0/16. PrivateLink scales as services × consumer VPCs—10 services consumed from 3 VPCs is 30 interface endpoints, roughly $438/month in endpoint hours before the provider-side NLBs. A service mesh buys mTLS at the price of a sidecar in every pod and a control plane someone has to own.

Amazon VPC Lattice solves the app-layer case: HTTP/HTTPS/gRPC calls across VPCs and accounts, IAM-authenticated per request, over a managed link-local data plane (169.254.171.0/24) that ignores overlapping CIDRs. But teams adopt it blind and hit real walls: no WebSockets, no UDP, a 10-minute maximum connection lifetime, 10 Gbps per service per AZ. This kit makes the AI assistant decide honestly first, then generate production-ready Terraform with auth policies, observability, and a verification script that proves the deny-by-default posture actually denies.

## Who This Is For

- Platform engineers at startups with 5–50 microservices spread across 2+ accounts who were just told "add Istio" and would rather not
- Teams blocked by overlapping VPC CIDRs after an acquisition or environment cloning
- Anyone exposing internal APIs across team/account boundaries who needs caller identity enforced at the network, not just inside app code

## How to Use

1. Open Kiro CLI, Claude Code, or Amazon Q Developer CLI in an empty workspace with AWS credentials for both accounts (named profiles `provider` and `consumer` work well).
2. Paste the System Prompt below, replacing the bracketed placeholders: `[REGION]`, `[ACCOUNT_ID_A]`, `[ACCOUNT_ID_B]`, `[VPC_IDS]`, and the three example service names.
3. Let the assistant run its Reality Check First section—answer its questions about task roles, subnets, and RAM sharing. Do not let it skip to code generation.
4. Read `decision-table.md` first. If the verdict is TGW+ALB or PrivateLink for your traffic profile, stop—that is the kit working, and a far cheaper failure than after `terraform apply`.
5. Run `terraform init && terraform plan && terraform apply`, then `./verify.sh` and confirm 4 PASS lines.

**Prerequisites**

- **Required Access:** AdministratorAccess or PowerUserAccess plus `iam:PutRolePolicy` in both accounts; AWS RAM sharing enabled by the management account (`ram:EnableSharingWithAwsOrganization`); ability to create VPC Lattice resources (`vpc-lattice:*`).
- **Recommended Background:** VPC fundamentals, what SigV4 request signing is, Terraform basics.
- **Tools Required:** Terraform >= 1.5 with AWS provider >= 5.0, AWS CLI v2, AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) for quota/AZ verification, and the AWS API MCP server or Kiro's `use_aws` tool for the live reality checks.

**Key Parameters:** Auth type (AWS_IAM), listener (HTTPS 443), health check (/healthz, 30s interval, 2 healthy/3 unhealthy), access-log retention (90d), budget ($250/mo, alerts at 80%/100%), blue/green target-group weights (90/10), region (us-east-1).

**Troubleshooting:** If curl from a client VPC times out even though DNS resolves to a 169.254.171.x address, it is expected when the client security group lacks egress 443 to the VPC Lattice managed prefix list (`com.amazonaws.us-east-1.vpc-lattice`)—Lattice traffic never originates from the provider VPC's CIDR, so legacy CIDR-based egress rules silently fail.

## System Prompt

```
# VPC Lattice Service Network — Architecture & Implementation Request

## Project Overview
Design and generate production-ready Terraform for application-layer service-to-service
connectivity across 3 VPCs in 2 AWS accounts ([ACCOUNT_ID_A], [ACCOUNT_ID_B], region
[REGION]) using Amazon VPC Lattice. Two VPCs overlap on 10.0.0.0/16, so network-layer
routing (peering, Transit Gateway) requires re-IP and is off the table. Services:
`orders` (ECS Fargate, IP targets, port 8080), `payments` (Lambda), `inventory`
(internal ALB target). Every call must be IAM-authenticated with SigV4. No service
mesh, no sidecar proxies for routing.

## Reality Check First — do this before generating anything
1. Verify VPC Lattice is available in [REGION] and that no workload subnet sits in an
   unsupported AZ (use1-az3, usw1-az2, apne1-az3, apne2-az2, euw1-az4, cac1-az3,
   ilc1-az2). Check with aws_read_documentation and `aws ec2 describe-availability-zones`.
2. Confirm each client VPC has no existing service network association — a VPC supports
   exactly 1 (use VPC endpoints of type service-network if more are needed).
3. Confirm AWS RAM organization sharing is enabled before planning cross-account
   (`aws ram get-resource-share-associations`).
4. Confirm the ECS task role and Lambda execution role ARNs exist (`aws iam get-role`).
   Auth policies must reference real principals — never "Principal": "*" with no conditions.
5. If any check fails, STOP and report the blocker with exact CLI output. Do not
   generate Terraform against unverified assumptions.

## Detailed Requirements
### 1. Service Network & Sharing
- One service network `shared-services` with auth_type AWS_IAM. Wire it with
  aws_vpclattice_service_network, aws_vpclattice_service_network_vpc_association (x3),
  aws_vpclattice_service_network_service_association (x3).
- Share the service network to [ACCOUNT_ID_B] via AWS RAM (aws_ram_resource_share +
  principal association). Attach a security group to each VPC association allowing 443
  only from workload subnets.

### 2. IAM Auth — deny-by-default
- Service network auth policy: allow only authenticated principals from the two account
  IDs; anonymous traffic is implicitly denied.
- Per-service auth policy on `payments` (aws_vpclattice_auth_policy): only the orders
  task role may invoke POST on /charge. Use action vpc-lattice-svcs:Invoke and
  condition keys vpc-lattice-svcs:RequestMethod and vpc-lattice-svcs:SourceVpc. Keep
  every policy under the 10 KB auth policy limit.
- Client side: SigV4 signing with service name vpc-lattice-svcs. Payload signing is not
  supported — send x-amz-content-sha256: UNSIGNED-PAYLOAD. Provide an aws-sigv4-proxy
  sidecar task definition for the one legacy client that cannot sign natively.

### 3. Routing & Targets
- HTTPS listeners on 443 (max 2 listeners/service, 10 rules/listener — design within
  these). Target groups: type IP for ECS, LAMBDA for payments, ALB for inventory
  (1,000 targets/group max). Health checks: /healthz, 30s interval, 2 healthy /
  3 unhealthy thresholds (Lambda target groups have no health checks — note this).
- Weighted routing 90/10 between blue and green target groups for `orders`.

### 4. Observability & Cost Controls
- aws_vpclattice_access_log_subscription per service to CloudWatch Logs, 90-day
  retention. Alarms: HTTPCode_5XX_Count > 1% of requests over 5 min, and p99
  TotalResponseTime. AWS Budgets: $250/month with 80% and 100% alerts.
- Emit decision-table.md comparing this exact topology at 1 TB/month: Lattice
  ($0.025/service-hr + $0.025/GB + $0.10 per 1M requests after the free 300,000
  requests/hour) vs TGW+internal ALB ($0.05/attachment-hr + $0.02/GB + ALB hours/LCU)
  vs PrivateLink ($0.01/AZ-hr per endpoint + $0.01/GB). Show monthly totals and name
  the winner for: many-to-many internal APIs, one high-throughput service, non-HTTP.

### 5. Honest Constraints Gate
Fail the design loudly — in limits.md and in chat — if any workload needs: WebSockets
or UDP (unsupported), raw TCP (only via VPC Lattice resource gateways/resource
configurations, not services), connections longer than the 10-minute maximum lifetime
or 1-minute idle timeout, more than the default 10 Gbps per service per AZ or 10,000
requests/sec per service per AZ, cross-Region calls (Lattice is Region-scoped), or
direct on-premises callers (Lattice answers on link-local 169.254.171.0/24, unreachable
over DX/VPN without a proxy fleet inside an associated VPC).

## Deliverables Requested (Terraform preferred, no preamble)
1. lattice.tf — service network, services, listeners, target groups, associations
2. auth-policies/*.json + aws_vpclattice_auth_policy wiring
3. ram-share.tf — cross-account sharing
4. observability.tf — access logs, alarms, budget
5. client-sigv4/ — SDK signing snippet + aws-sigv4-proxy task definition
6. decision-table.md — priced comparison with a one-line verdict
7. limits.md — every constraint above, mapped to this workload
8. verify.sh — acceptance evidence

## Acceptance Evidence
verify.sh must prove, printing PASS/FAIL per check: (1) unsigned curl to payments
returns 403 AccessDeniedException; (2) a SigV4-signed POST /charge from the orders task
role returns 200; (3) a signed DELETE is denied; (4) an access-log line lands in
CloudWatch Logs within 2 minutes. The full stack must `terraform apply` cleanly in
under 15 minutes.
```

## What You Get

1. `lattice.tf` — service network (AWS_IAM auth), 3 services, HTTPS listeners, IP/LAMBDA/ALB target groups, all VPC and service associations
2. `auth-policies/` — service-network policy + per-service `payments` policy as JSON, wired via `aws_vpclattice_auth_policy`
3. `ram-share.tf` — AWS RAM share of the service network to the consumer account
4. `observability.tf` — access-log subscriptions (90d CloudWatch Logs), 5XX and p99 latency alarms, $250/mo budget with 80%/100% alerts
5. `client-sigv4/` — SigV4 signing code snippet plus an aws-sigv4-proxy sidecar task definition for non-signing clients
6. `decision-table.md` — Lattice vs TGW+ALB vs PrivateLink with real us-east-1 prices and a verdict for your topology
7. `limits.md` — the honest "Lattice cannot do this" list mapped to your workloads
8. `verify.sh` — 4-check acceptance script proving deny-by-default works

## Example Output

```json
// auth-policies/payments.json (excerpt)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::222222222222:role/orders-task-role" },
    "Action": "vpc-lattice-svcs:Invoke",
    "Resource": "arn:aws:vpc-lattice:us-east-1:111111111111:service/svc-0a1b2c3d4e5f67890/*",
    "Condition": {
      "StringEquals": {
        "vpc-lattice-svcs:RequestMethod": "POST",
        "vpc-lattice-svcs:SourceVpc": "vpc-0abc1234def567890"
      }
    }
  }]
}
```

```
$ ./verify.sh
[PASS] unsigned curl -> 403 AccessDeniedException (anonymous denied)
[PASS] SigV4 POST /charge as orders-task-role -> 200
[PASS] SigV4 DELETE /charge -> 403 (method not in policy)
[PASS] access log visible in CloudWatch Logs after 41s
```

## AWS Services Used

Amazon VPC Lattice, AWS Identity and Access Management (IAM), AWS Resource Access Manager (RAM), Amazon CloudWatch, AWS Budgets, Amazon ECS, AWS Lambda, Elastic Load Balancing (ALB targets), Amazon VPC

## Well-Architected Alignment

- **Security:** deny-by-default IAM auth policies; SigV4-verified caller identity replaces IP-based trust; security groups pinned on every VPC association; no `Principal: "*"` without conditions.
- **Reliability:** health checks (30s, 2/3 thresholds), weighted blue/green routing, per-AZ throughput/RPS quotas surfaced *before* deployment so capacity ceilings are a design input, not an outage.
- **Performance Efficiency:** managed data plane removes the per-pod sidecar hop and mesh control plane; MTU 8500 noted for large payloads.
- **Cost Optimization:** priced three-way decision table is a mandatory deliverable; $250/mo budget with 80%/100% alerts ships in the same apply; 300K req/hr free tier factored in.
- **Operational Excellence:** verify.sh emits acceptance evidence; access logs with 90-day retention and named alarms from day one.

## Cost Notes

us-east-1, June 2026, 730-hour month. Worked example: 10 services consumed from 3 VPCs, 1 TB/month total.

| Dimension | VPC Lattice | TGW + internal ALB | PrivateLink |
|---|---|---|---|
| Hourly | $0.025/service-hr ($18.25/mo per service); VPC associations free | $0.05/attachment-hr per VPC ($36.50/mo each) + ALB $0.0225/hr + $0.008/LCU-hr | $0.01/AZ-hr per endpoint ($14.60/mo per 2-AZ endpoint) + provider NLB $0.0225/hr |
| Per-GB | $0.025/GB + $0.10/1M requests (first 300K req/hr free) | $0.02/GB (TGW data processing) + LCU | $0.01/GB (first 1 PB) |
| Built-in auth | IAM auth policies, SigV4 caller identity | None — SGs + auth you build | None — SGs only |
| Overlapping CIDRs | Fine (link-local 169.254.171.0/24 data plane) | Breaks — re-IP required | Fine |
| Protocols | HTTP/1.1, HTTP/2, gRPC, TLS passthrough; TCP only via resource gateways; no UDP, no WebSockets | Any IP protocol | TCP via NLB |
| Scales as | N services (flat) | N VPC attachments | N services × M consumer VPCs |

- **Lattice:** 10 × $0.025 × 730 = $182.50 + 1,024 GB × $0.025 = $25.60 + $0 requests (under free tier) ≈ **$208/mo**
- **TGW + ALB:** 4 attachments × $0.05 × 730 = $146.00 + $20.48 data + $16.43 ALB + ~$11.68 LCU ≈ **$195/mo** — cheaper on paper, but impossible here without re-IP, and you still build authn yourself
- **PrivateLink:** 30 endpoints × 2 AZ × $0.01 × 730 = $438.00 + $10.24 data + 10 NLBs × $16.43 ≈ **$612/mo**

Honest flip side: per-GB, Lattice ($0.025) is 2.5× PrivateLink ($0.01). One chunky service pushing 20 TB/month costs $512 in Lattice data processing vs ~$219 total on PrivateLink — for single high-throughput producer/consumer pairs, PrivateLink wins. The kit's decision table calls this out automatically.

## Troubleshooting

1. **403 AccessDeniedException on correctly signed requests** — Cause: payload was signed, or the wrong signing service name was used. Fix: sign with service name `vpc-lattice-svcs` and send `x-amz-content-sha256: UNSIGNED-PAYLOAD`.
2. **DNS resolves to 169.254.171.x but connections time out** — Cause: client SG has no egress 443 to the VPC Lattice managed prefix list, or the VPC association's security group blocks the client subnet. Fix: add egress to `com.amazonaws.<region>.vpc-lattice` and open the association SG to workload subnets.
3. **`terraform apply` fails associating a VPC with a second service network** — Cause: hard limit of 1 service network association per VPC. Fix: consolidate into one service network, or use VPC endpoints of type service-network (200/service network) for multi-network access.
4. **gRPC streams drop at exactly 10 minutes** — Cause: VPC Lattice's maximum connection lifetime (10 min; idle timeout 1 min). Fix: add reconnect/retry logic; move genuinely long-lived streaming hops to NLB or PrivateLink.
5. **Shared service network invisible in the consumer account** — Cause: RAM invitation not accepted or organization sharing disabled. Fix: enable sharing from the management account, then `aws ram get-resource-share-invitations` and accept.
