# Private VPC Access Without a Public Bastion: Client VPN vs Tailscale vs SSM Decision Kit

Compares the three real ways a small team reaches private RDS, internal dashboards, and EC2 shells—then deploys the cheapest one that fits, with per-team-size cost math, split-tunnel config, and SSO wired in—so you stop paying $115/month for a VPN nobody needed, or exposing port 22 to the internet.

## The Problem

Every startup hits this in month one: the database is in a private subnet (correctly), and now nobody can run `psql`. The two default answers are both wrong. Answer one is a public bastion: a t3.micro with port 22 open to "the office IP," which drifts to 0.0.0.0/0 within a quarter, and GuardDuty starts firing `UnauthorizedAccess:EC2/SSHBruteForce` findings while you hold a long-lived `.pem` file nobody rotates. Answer two is reflexively standing up AWS Client VPN without doing the math: the subnet association alone bills $0.10/hour, 730 hours a month, every month—a **$73.00/month floor before a single person connects**. Add connection-hours at $0.05/hour and a 5-person team working 8-hour days pays about **$115/month**. Meanwhile, for a team that only needs Postgres on 5432 and an internal admin UI on 8080, SSM Session Manager port forwarding delivers the same access for **$0.00**, fully logged in CloudTrail, with zero new infrastructure. The third path, a Tailscale subnet router on a $3.07/month t4g.nano, sits in between and wins when you need full network access with consumer-grade UX. Most teams never see this comparison—they pick whatever the last blog post showed and overpay or overexpose for years.

## Who This Is For

- 2–25 person startup engineering teams whose VPC has private subnets and no sanctioned way in
- The platform engineer who inherited a public bastion and needs a defensible replacement by Friday
- Founders deciding whether a third-party mesh VPN is acceptable, or whether compliance requires staying AWS-native
- Anyone about to click "Create Client VPN endpoint" who has not yet computed the $73/month idle floor

## How to Use

1. Copy the System Prompt below into Kiro CLI, Claude Code, or Amazon Q Developer CLI in a workspace where `aws sts get-caller-identity` succeeds against the target account.
2. Replace the bracketed placeholders: `[TEAM_SIZE]`, `[VPC_ID]`, `[VPC_CIDR]`, `[REGION]`, `[BUDGET]`, and the list of resources you need to reach.
3. Let the assistant run its read-only verification calls (`aws ec2 describe-vpcs`, `aws ssm describe-instance-information`) and confirm its findings before anything is generated. Do not skip this—it is what keeps the recommendation honest.
4. Review `decision-table.md` first. If you accept the recommendation, apply `main.tf` with `terraform init && terraform plan && terraform apply`.
5. Run `verify.sh`, then onboard one teammate using `onboarding.md` and time it against the under-10-minute bar.

**Prerequisites**

- **Required Access:** Read-only `ec2:Describe*` and `ssm:DescribeInstanceInformation` for the evaluation phase; for deployment, permissions to create Client VPN resources (`ec2:CreateClientVpnEndpoint`, `ec2:AssociateClientVpnTargetNetwork`, `ec2:AuthorizeClientVpnIngress`), `acm:ImportCertificate`, EC2 launch rights, and IAM Identity Center admin if wiring SAML SSO.
- **Recommended Background:** Basic VPC routing and security groups; Terraform 1.5+; familiarity with `aws sso login`.
- **Tools Required:** AWS CLI v2 with the Session Manager plugin installed; Terraform >= 1.5; the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) for the pricing and limits checks the prompt mandates; a Tailscale account only if that path wins.

**Key Parameters:** TEAM_SIZE (5), VPC_CIDR (10.0.0.0/16), REGION (us-east-1), BUDGET ($50/mo), Client VPN client CIDR (172.16.0.0/22, must not overlap VPC), work pattern for connection-hour math (8 hr x 21 days), session log retention (90d), split-tunnel (enabled), Tailscale router instance (t4g.nano).

**Troubleshooting:** If `verify.sh` reports `TargetNotConnected` on the SSM path, it's expected on instances with no outbound internet—Session Manager needs either a NAT route or the three interface VPC endpoints (`ssm`, `ssmmessages`, `ec2messages`). The prompt's verification step is designed to surface this before generation, not after.

## System Prompt

```markdown
# Private VPC Access Architecture Decision Request

## Project Overview
We are a [TEAM_SIZE]-person engineering team (default: 5) that needs private access to
resources inside VPC [VPC_ID] ([VPC_CIDR], default 10.0.0.0/16) in [REGION] (default
us-east-1): an RDS PostgreSQL endpoint on 5432, an internal admin UI on 8080, and shell
access to EC2 instances. We refuse to run a public bastion host. Evaluate exactly three
production-ready options and implement the winner: (1) AWS Client VPN, (2) a Tailscale
subnet router on EC2, (3) AWS Systems Manager Session Manager port forwarding.

## Verify Reality First (before generating anything)
- Run aws ec2 describe-vpcs and describe-subnets to confirm [VPC_ID], its CIDR, and at
  least one private subnet exist. Report what you found.
- Run aws ssm describe-instance-information and confirm target instances report
  PingStatus Online with SSM Agent 3.x. If any are offline, option 3 needs remediation
  first—say so explicitly instead of assuming.
- Confirm current AWS Client VPN pricing for [REGION] via aws_read_documentation against
  aws.amazon.com/vpn/pricing (us-east-1 baseline: $0.10/hr per subnet association,
  $0.05 per connection-hour). Never invent prices.
- Confirm the proposed Client VPN client CIDR (must be between /22 and /12) does not
  overlap [VPC_CIDR] or any peered CIDR.

## Detailed Requirements
### 1. Decision Table
One markdown table, three columns. Rows: monthly cost at team sizes 3 / 5 / 10 / 25
with the math shown (Client VPN base = 730 hr x $0.10 subnet association = $73.00/mo
even at zero usage; connection-hours = users x 8 hr x 21 workdays x $0.05; Tailscale =
plan seats + one t4g.nano subnet router at ~$3.07/mo; SSM = $0.00, or ~$21.90/mo only
if three interface endpoints are required), SSO options, full-L3 vs per-port access,
audit trail location, client software required, setup time, third-party dependency.
Close with a 4-sentence recommendation specific to [TEAM_SIZE].
### 2. Cost Target
Keep total access spend under $[BUDGET]/month (default $50). If every listed resource
is a reachable TCP port, state explicitly that SSM port forwarding
(AWS-StartPortForwardingSessionToRemoteHost) is sufficient and skip both VPNs.
### 3. Security & Compliance
Zero security-group rules open to 0.0.0.0/0 anywhere in the design. SSO wired in for
whichever path wins: SAML 2.0 federation through IAM Identity Center plus the
self-service portal for Client VPN; Google / Okta / Microsoft Entra ID / GitHub IdP
for Tailscale; aws sso login short-lived credentials for SSM. All sessions and
connections logged to CloudWatch Logs with 90-day retention.
### 4. Implementation
Terraform preferred. Generate the winning option in full, plus the SSM fallback script
regardless of winner. Client VPN MUST be created with --split-tunnel so only
[VPC_CIDR] routes through the tunnel and internet traffic never hairpins through NAT.
The Tailscale router MUST advertise only [VPC_CIDR] (tailscale up
--advertise-routes=[VPC_CIDR]), set net.ipv4.ip_forward=1, and disable the ENI
source/dest check.

## Deliverables Requested
1. decision-table.md — the 3-way comparison with cost math shown
2. main.tf — the winning option, all resources tagged Environment and Owner
3. ssm-port-forward.sh — parameterized wrapper around
   AWS-StartPortForwardingSessionToRemoteHost for RDS and the admin UI
4. onboarding.md — the 10-minute new-teammate connection guide
5. verify.sh — acceptance evidence: psql reaches RDS by private DNS through the new
   path; an aws ec2 describe-security-groups query proves zero 0.0.0.0/0 ingress
   rules; the test connection appears in CloudWatch Logs

## Output Format
Map each option to the Well-Architected Security and Cost Optimization pillars in one
line each. Start directly with the decision table, without any preamble. Success bar:
a brand-new teammate gets a working connection in under 10 minutes.
```

## What You Get

1. **decision-table.md** — the 3-way Client VPN / Tailscale / SSM comparison with monthly cost computed at team sizes 3, 5, 10, and 25, plus a recommendation paragraph specific to your team
2. **main.tf** — production-ready Terraform for the winning path (Client VPN endpoint with SAML auth and split-tunnel, or the Tailscale subnet-router EC2 instance, or the SSM IAM/logging baseline), tagged and logged
3. **ssm-port-forward.sh** — a wrapper script around `AWS-StartPortForwardingSessionToRemoteHost` so `psql -h localhost -p 5432` just works
4. **onboarding.md** — the guide a new hire follows to get connected in under 10 minutes
5. **verify.sh** — acceptance evidence: a live connection test, a security-group audit proving no public ingress, and a CloudWatch Logs check confirming the session was recorded

## Example Output

> ### Recommendation for a 5-person team
> | | AWS Client VPN | Tailscale on EC2 | SSM Port Forwarding |
> |---|---|---|---|
> | Monthly cost (5 users) | **$115.00** ($73.00 association + $42.00 connection-hrs) | **$33.07** (5 x $6 Starter + $3.07 t4g.nano) | **$0.00** |
> | SSO | SAML 2.0 / IAM Identity Center | Google, Okta, Entra ID, GitHub | `aws sso login` |
> | Access model | Full L3 to VPC CIDR | Full L3 via advertised route | Per TCP port |
> | Audit trail | CloudWatch Logs connection logs | Tailscale admin console | CloudTrail + Session logs |
>
> Your listed resources are two TCP ports. **SSM port forwarding meets the requirement at $0.00/month**—deploy the VPN only when you need full network access for contractors without IAM identities.

## AWS Services Used

AWS Client VPN, AWS Systems Manager (Session Manager), Amazon EC2, Amazon VPC, AWS Certificate Manager, AWS IAM Identity Center, Amazon CloudWatch Logs, AWS CloudTrail

## Well-Architected Alignment

- **Security:** Eliminates public ingress entirely—zero 0.0.0.0/0 rules, no long-lived SSH keys; every path authenticates through SSO (SAML/IAM Identity Center, IdP, or short-lived `aws sso login` credentials) and every session lands in CloudWatch Logs or CloudTrail with 90-day retention.
- **Cost Optimization:** The decision is driven by computed monthly cost per team size, not habit; the prompt explicitly defaults to the $0.00 SSM path when per-port access suffices, and split-tunnel prevents paying NAT data-processing fees ($0.045/GB) on hairpinned internet traffic.
- **Operational Excellence:** `verify.sh` produces evidence the design works before anyone calls it done; `onboarding.md` makes access a 10-minute self-service task instead of tribal knowledge.
- **Reliability:** Client VPN and SSM are managed, multi-AZ-capable services; the kit flags that a second Client VPN subnet association doubles the floor to $146/month so you make the HA trade-off consciously.

## Cost Notes

Us-east-1, 8 hr/day x 21 workdays usage pattern:

| Team size | AWS Client VPN | Tailscale + t4g.nano router | SSM port forwarding |
|---|---|---|---|
| 3 | $73.00 + $25.20 = **$98.20/mo** | $0 (Personal plan, 3 users) + $3.07 = **$3.07/mo** | **$0.00** |
| 5 | $73.00 + $42.00 = **$115.00/mo** | $30.00 + $3.07 = **$33.07/mo** | **$0.00** |
| 10 | $73.00 + $84.00 = **$157.00/mo** | $60.00 + $3.07 = **$63.07/mo** | **$0.00** |
| 25 | $73.00 + $210.00 = **$283.00/mo** | $150.00 + $3.07 = **$153.07/mo** | **$0.00** (UX strains; consider a VPN) |

Client VPN math: subnet association $0.10/hr x 730 hr = $73.00 fixed (billed even at zero connections; a second AZ association doubles it), plus $0.05 per connection-hour. Tailscale Starter is $6/user/month; the free Personal plan covers up to 3 users. SSM Session Manager itself is free; if instances lack outbound internet, three interface VPC endpoints add ~$21.90/month ($0.01/hr each) plus $0.01/GB processed.

## Troubleshooting

1. **`TargetNotConnected` when starting an SSM session** — Cause: the instance isn't registered with Systems Manager (missing the `AmazonSSMManagedInstanceCore` managed policy on its instance profile, or no path to SSM endpoints). Fix: attach the instance profile, add a NAT route or the `ssm`/`ssmmessages`/`ec2messages` interface endpoints, then restart `amazon-ssm-agent`.
2. **Client VPN connects but nothing in the VPC responds** — Cause: authorization rules are separate from routes and default to none. Fix: `aws ec2 authorize-client-vpn-ingress --target-network-cidr <VPC_CIDR> --authorize-all-groups` and confirm the endpoint route table covers the subnet.
3. **All internet traffic crawls after connecting to Client VPN** — Cause: split-tunnel is off, so every byte hairpins through the VPC and its NAT gateway at $0.045/GB. Fix: enable `--split-tunnel` on the endpoint so only the VPC CIDR rides the tunnel.
4. **Tailscale clients can't reach VPC IPs despite the router being up** — Cause: advertised routes await approval in the admin console, IP forwarding is off, or the ENI source/dest check is dropping forwarded packets. Fix: approve routes, set `net.ipv4.ip_forward=1`, and run `aws ec2 modify-instance-attribute --no-source-dest-check`.
5. **A $73 Client VPN line item with zero usage** — Cause: subnet association bills 24/7 whether anyone connects. Fix: `disassociate-client-vpn-target-network` for idle environments, or replace with the $0 SSM path this kit generates.
