# Kill the Public IPv4 Tax: VPC Dual-Stack Migration Kit with Egress-Only IGW

Inventories every billable public IPv4 in your VPC, then generates a gated dual-stack migration — Terraform, a phase-by-phase runbook, and an honest list of what still needs IPv4 — so the $3.65-per-IP line item disappears without a single dropped connection.

## The Problem

On February 1, 2024, AWS started charging **$0.005/hour for every public IPv4 address** — in use or idle, EIP or auto-assigned. That is **$3.65/month ($43.80/year) per address**. A typical 3-AZ production VPC built before 2024 carries: an internet-facing ALB (3 IPs), 3 NAT gateway EIPs, 4 instances with `MapPublicIpOnLaunch` auto-assigned IPs, and 2 forgotten idle EIPs — **12 addresses = $43.80/month = $525.60/year**, before a single byte of traffic. Multiply by dev/staging/prod and you're paying ~$1,576/year for *numbers*. Worse, the charge hides under "Amazon Virtual Private Cloud" in the bill — a line most teams never expand.

IPv6 addresses are free. The egress-only internet gateway costs $0.00/hour with no per-GB processing fee. But teams stall on the migration because the failure modes are real: publish an AAAA record before your security groups allow IPv6 and you blackhole every IPv6-capable client; strip IPv4 before checking your third-party dependencies and your CI can no longer reach github.com (which still has no AAAA records). This prompt sequences the migration so each phase is verified before the next begins — and tells you honestly what must stay on IPv4.

## Who This Is For

Startups and platform engineers running VPCs designed before the 2024 pricing change; anyone whose "Amazon Virtual Private Cloud" bill line jumped in February 2024; teams who want the savings but need a production-safe rollout order instead of a weekend of trial-and-error.

## How to Use

1. Open Kiro CLI, Claude Code, or Amazon Q Developer CLI in a session with AWS credentials for the target account (read access minimum for the audit phase).
2. Copy the System Prompt below into the chat (or save as `.kiro/steering/ipv4-elimination.md` for Kiro).
3. Replace the bracketed placeholders: `[VPC_ID]`, `[REGION]`, and `[STACK_SERVICES]` (the services your workload actually uses, e.g., "ALB, EC2 Auto Scaling, ECS on Fargate, RDS Postgres, S3").
4. Let the assistant run the Verify Reality First checks — it must show you the audit table and current monthly spend before generating any Terraform.
5. Apply Phase 1 (plumbing — zero traffic impact), run `verify-ipv6.sh`, then proceed phase by phase. Do not skip the 7-day soak gate before Phase 4.

**Prerequisites**

- **Required Access:** `ec2:Describe*`, `ec2:AssociateVpcCidrBlock`, `ec2:AssociateSubnetCidrBlock`, `ec2:CreateEgressOnlyInternetGateway`, `ec2:ModifySubnetAttribute`, `elasticloadbalancing:SetIpAddressType`, `route53:ChangeResourceRecordSets`, `ce:GetCostAndUsage`. Run in a sandbox account first if you have one.
- **Recommended Background:** Basic VPC routing (route tables, IGW vs NAT), how ALB target groups and Route 53 alias records work, Terraform 1.5+.
- **Tools Required:** AWS CLI v2, Terraform >= 1.5, AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) so the assistant verifies current per-service IPv6 support instead of reciting stale training data.

**Key Parameters:** IPv6 CIDR (/56 Amazon-provided), one /64 per subnet, AAAA record TTL (60s during cutover, 300s steady-state), soak gate before IPv4 removal (7 days), DNS64 (enabled on private subnets), NAT64 route (64:ff9b::/96 → one retained NAT gateway), savings target (>=80% of public IPv4 charges eliminated in 30 days).

**Troubleshooting:** If the generated plan keeps one NAT gateway, that is expected — IPv4-only destinations like github.com (no AAAA records) still need a NAT64/DNS64 path. The egress-only internet gateway carries native IPv6 only; it cannot translate to IPv4.

## System Prompt

```
# Public IPv4 Elimination — Dual-Stack Migration Plan Request

## Project Overview
Act as a senior AWS network engineer. My production VPC ([VPC_ID] in [REGION]) pays the $0.005/hour public IPv4 charge (in effect since February 1, 2024) on every public address — ALB nodes, NAT gateway EIPs, auto-assigned instance IPs, and idle EIPs. Stack: [STACK_SERVICES]. Produce a production-ready, phased dual-stack (IPv4 + IPv6) migration that removes every public IPv4 we do not strictly need, without dropping a single connection.

## Verify Reality First — Error Management (do this before generating anything)
1. Run `aws sts get-caller-identity` and confirm account and [REGION] with me before touching anything.
2. Run `aws ec2 describe-vpcs --vpc-ids [VPC_ID]` — if an IPv6 CIDR is already associated, plan around it; do not allocate a second one. If `associate-vpc-cidr-block` would exceed the quota of 5 IPv6 CIDR blocks per VPC, report existing associations instead of retrying.
3. Pull last month's Cost Explorer data filtered to usage types `PublicIPv4:InUseAddress` and `PublicIPv4:IdleAddress`; report the exact current monthly spend.
4. Inventory every public IPv4 via VPC IPAM Public IP Insights (free) or `aws ec2 describe-network-interfaces --filters Name=association.public-ip,Values=*`, mapped to its owning resource.
5. For every service in [STACK_SERVICES], confirm current dual-stack and IPv6-only support against the official "AWS services that support IPv6" documentation page using the aws_read_documentation MCP tool. Never assert IPv6 support from memory — coverage changes frequently and per-service support differs between dual-stack and IPv6-only.
If any check fails or returns empty, STOP and report what you found instead of guessing.

## Detailed Requirements
### 1. Cost Audit
- Per-IP math at $0.005/hr x 730 hrs = $3.65/month, $43.80/year; subtotal per resource class (ALB, NAT EIPs, instances, idle EIPs).
- Flag idle EIPs for immediate release — day-0 savings, zero risk.
### 2. Dual-Stack Plumbing (zero traffic impact)
- Associate an Amazon-provided /56 (`aws ec2 associate-vpc-cidr-block --amazon-provided-ipv6-cidr-block`); carve one /64 per subnet via `associate-subnet-cidr-block`.
- Routes: `::/0 -> internet gateway` in public route tables; `::/0 -> egress-only internet gateway` ($0.00/hour, no per-GB fee) in private route tables. Never route private-subnet IPv6 at a NAT gateway.
- Add explicit IPv6 security-group and NACL rules mirroring existing IPv4 intent. Never add blanket `::/0` inbound.
### 3. Rollout Order (strict, gated)
- Phase 1 plumbing -> Phase 2 edge: `aws elbv2 set-ip-address-type --ip-address-type dualstack`, then Route 53 AAAA alias records at TTL 60 only after `curl -6` against the ALB succeeds -> Phase 3 compute: `modify-subnet-attribute --assign-ipv6-address-on-creation`, launch-template Ipv6AddressCount=1 -> Phase 4 strip IPv4: ALB to `dualstack-without-public-ipv4`, `--no-map-public-ip-on-launch`, release idle EIPs.
- Hard gate: 7 consecutive days of healthy dual-stack metrics (ALB HealthyHostCount per AZ, HTTPCode_ELB_5XX_Count unchanged) before ANY IPv4 removal. Every phase gets an explicit rollback command.
- DNS64/NAT64 escape hatch for IPv4-only destinations: `modify-subnet-attribute --enable-dns64` plus a `64:ff9b::/96` route to one retained NAT gateway.
### 4. Honest "Still Needs IPv4" List
- Enumerate everything in my stack that cannot go IPv6-only today, each with verification evidence: third-party endpoints with no AAAA records (test with `dig AAAA`, e.g. github.com), data stores that support dual-stack but not IPv6-only (e.g., RDS), non-Nitro instance types barred from IPv6-only subnets, SMTP egress deliverability. No hand-waving — every entry cites a doc page or a command output.
### 5. Cost Target
- Eliminate >=80% of public IPv4 charges within 30 days. A before/after monthly cost table is a required deliverable.

## Deliverables Requested (Terraform preferred)
1. `ipv4-audit.md` — every public IP, owning resource, monthly cost, disposition (keep / migrate / release)
2. `dualstack.tf` — VPC IPv6 CIDR association, subnet /64s, egress-only IGW, routes, IPv6 SG rules
3. `runbook.md` — phase-by-phase commands with a rollback step per phase
4. `still-ipv4.md` — the honest list with evidence
5. `verify-ipv6.sh` — curl -6, dig AAAA, and route checks proving each phase works before the next begins

## Acceptance Evidence
End with a verification section: the exact commands whose output proves dual-stack works (curl -6 against the ALB DNS name, dig AAAA on the public record, an EIGW route check from a private instance) plus the projected new monthly bill.

Align with the Well-Architected Cost Optimization and Reliability pillars. Output the artifacts directly without preamble. Phase 1 must apply in under 15 minutes with zero data-plane disruption.
```

## What You Get

1. **`ipv4-audit.md`** — every billable public IPv4 in the VPC mapped to its owning resource, with per-IP and total monthly cost and a keep/migrate/release disposition.
2. **`dualstack.tf`** — Terraform for the IPv6 /56 association, per-subnet /64s, egress-only internet gateway, `::/0` routes, and explicit IPv6 security-group rules.
3. **`runbook.md`** — the 4-phase rollout with exact CLI commands, the 7-day soak gate, and a rollback command for every phase.
4. **`still-ipv4.md`** — the honest list: what in *your* stack cannot do IPv6 yet, each entry backed by a `dig AAAA` output or an AWS doc citation.
5. **`verify-ipv6.sh`** — acceptance checks (curl -6, dig AAAA, EIGW route test) that gate each phase.
6. A before/after cost table projecting the new "Amazon Virtual Private Cloud" bill line.

## Example Output

```
ipv4-audit.md (excerpt)
| Public IP      | Resource                  | $/month | Disposition            |
|----------------|---------------------------|---------|------------------------|
| 54.210.x.x x3  | ALB app/web-prod (3 AZs)  | $10.95  | Phase 4: dualstack-without-public-ipv4 |
| 3.218.x.x x3   | NAT gateway EIPs          | $10.95  | Keep 1 for NAT64, release 2 in Phase 4 |
| 44.192.x.x x2  | Idle EIPs (unassociated)  | $7.30   | Release TODAY — zero risk |
Current: $43.80/mo -> Projected after Phase 4: $7.30/mo (83% reduction)

runbook.md Phase 2 (excerpt)
aws elbv2 set-ip-address-type --load-balancer-arn $ALB_ARN --ip-address-type dualstack
curl -6 -sS https://$(aws elbv2 describe-load-balancers ... --query 'LoadBalancers[0].DNSName' --output text)  # MUST return 200 before AAAA publish
ROLLBACK: aws elbv2 set-ip-address-type --ip-address-type ipv4
```

## AWS Services Used

Amazon VPC (IPv6 CIDR, egress-only internet gateway, NAT gateway DNS64/NAT64), Amazon EC2, Elastic Load Balancing (ALB/NLB), Amazon Route 53, Amazon VPC IP Address Manager (IPAM) Public IP Insights, AWS Cost Explorer, Amazon CloudWatch, Amazon CloudFront (optional IPv6 origins).

## Well-Architected Alignment

- **Cost Optimization:** eliminates a recurring per-resource charge entirely instead of optimizing around it; idle-EIP release is day-0 savings; before/after cost table is a mandatory deliverable.
- **Reliability:** strict phase gates, AAAA published only after `curl -6` proof, 7-day dual-stack soak before any IPv4 removal, rollback command per phase.
- **Security:** explicit IPv6 security-group and NACL rules mirroring IPv4 intent — never blanket `::/0` inbound; egress-only IGW enforces outbound-only for private subnets by design.
- **Operational Excellence:** runbook + verification script as artifacts; reality checks via the AWS Documentation MCP server before generation.
- **Performance Efficiency:** native end-to-end IPv6 removes the NAT translation hop for migrated flows.

## Cost Notes

- Public IPv4: **$0.005/hour = $3.65/month = $43.80/year per address** (since Feb 1, 2024; applies in-use and idle; BYOIP addresses are exempt; AWS Free Tier covers 750 hrs/month of EC2 public IPv4 for the first 12 months of new accounts).
- Example fleet above: 12 addresses = **$43.80/month ($525.60/year)** dropping to $7.30/month after Phase 4 — an **83% reduction**, with one EIP retained for NAT64.
- Egress-only internet gateway: **$0.00/hour, $0.00/GB processing** — only standard data transfer rates apply. The NAT gateway it replaces for IPv6 egress costs $0.045/hour + $0.045/GB processed (us-east-1).
- VPC IPAM Public IP Insights is **free**; no IPAM Advanced tier required for the audit.

## Troubleshooting

1. **IPv6 clients time out after AAAA records go live.** Cause: security groups have no IPv6 rules — IPv4 rules never match IPv6 traffic. Fix: add IPv6 ingress on 80/443 to the ALB security group and check NACLs; then re-run `curl -6`.
2. **Private instances cannot reach the internet over IPv6.** Cause: the private route table's `::/0` points at a NAT gateway (or is missing) — NAT gateways do not forward native IPv6. Fix: `aws ec2 create-egress-only-internet-gateway` and route `::/0` at the `eigw-` ID.
3. **Instance fails to launch in an IPv6-only subnet.** Cause: non-Nitro instance type (e.g., t2) — IPv6-only subnets require Nitro. Fix: move to t3/t4g/m5-class or keep that tier dual-stack.
4. **Switching the ALB to `dualstack-without-public-ipv4` breaks CloudFront origin fetches.** Cause: the distribution's origin connectivity is still IPv4 (the default). Fix: set origin connectivity to dualstack (supported since September 2025) or keep public IPv4 on that ALB.
5. **The bill barely drops after migration.** Cause: idle EIPs still allocated, or public IPs in other Regions/accounts. Fix: `aws ec2 describe-addresses` per Region, release unassociated EIPs, and group Cost Explorer by Region on usage type `PublicIPv4:IdleAddress`.
