# Multi-VPC Topology Decision Kit: Transit Gateway vs Peering vs PrivateLink Cost-Crossover Analyzer

Pick the right inter-VPC connectivity—with the exact GB/month where Transit Gateway stops being cheaper than peering—so you connect environments without quietly burning $36/mo per attachment plus $0.02/GB on traffic full-mesh peering would have moved for free.

## The Problem

A startup splits prod, staging, and shared-services across three VPCs and reaches for Transit Gateway because a blog said "use TGW for multi-VPC." TGW bills **$0.05/hr per VPC attachment (~$36/mo each)** plus a **$0.02/GB data processing fee on every GB**. Three attachments = **$108/mo before a single byte moves**. If those VPCs exchange 5 TB/month, that is another **$100/mo** in TGW processing. The same three-VPC mesh over **VPC peering costs $0/mo for the connections** and **$0/GB for traffic that stays in the same Availability Zone** (cross-AZ peering is $0.01/GB each way). The team is paying **$208/mo** for something that could cost near zero—or, worse, they peer everything and hit the **125-peering-per-VPC limit** and the no-transitive-routing wall six months later. Nobody computed the crossover. Nobody checked for overlapping CIDRs (10.0.0.0/16 in two VPCs = routing that silently breaks). This kit makes the AI do the math first.

## Who This Is For

- Founders and platform engineers running **2 or more VPCs** (multi-environment, multi-account, or post-acquisition) who need them to talk.
- Teams choosing between **Transit Gateway, VPC peering, and AWS PrivateLink** and tired of hand-wavy "it depends" answers.
- Anyone who has seen a surprise **data-processing line item** on the bill and wants the GB/month crossover number before committing.

## How to Use

1. Open Kiro CLI, Claude Code, or any AI assistant with the **AWS Documentation MCP server** connected so the model can verify current pricing and quotas.
2. Paste the System Prompt below into a new chat.
3. Fill the bracketed inputs: VPC count, account layout, region, AZ placement, expected GB/month per pair, and traffic direction (full mesh vs hub-spoke vs one-to-many service access).
4. Let the AI verify your real environment first (`aws ec2 describe-vpcs`, `describe-vpc-peering-connections`), then return the decision, the cost table, and the crossover GB/month.
5. Review the IP-overlap report and the chosen topology; ask for the Terraform or AWS CLI scaffold for the recommended option.

Prerequisites:
- Required Access: IAM permission for `ec2:DescribeVpcs`, `ec2:DescribeSubnets`, `ec2:DescribeVpcPeeringConnections`, `ec2:DescribeTransitGateways`, and read access to AWS Cost Explorer; the managed policy `ReadOnlyAccess` covers all of these.
- Recommended Background: basic CIDR/subnet math and the concept that VPC peering is non-transitive.
- Tools Required: AWS Documentation MCP server (`aws___read_documentation`, `aws___search_documentation`); AWS CLI v2 configured; optionally the AWS Pricing API.

Key Parameters: vpc_count (default 3), region (default us-east-1), same_az_assumption (default true → peering data $0/GB), gb_per_month_per_pair (default 5120), tgw_attachment ($0.05/hr ≈ $36/mo), tgw_data ($0.02/GB), peering_cross_az ($0.01/GB each way), privatelink_endpoint ($0.01/hr/AZ ≈ $7.30/mo), privatelink_data ($0.01/GB), peering_limit_per_vpc (125), tgw_attachment_quota (5000).

Troubleshooting: If the AI recommends PrivateLink for general VPC-to-VPC connectivity, that is wrong and expected when the prompt omits direction—PrivateLink is one-way service access (consumer reaches a single provider endpoint), not bidirectional network routing. Re-state that you need full mesh or hub routing and it will switch to peering or TGW.

## System Prompt

```
# Multi-VPC Connectivity Topology Decision Request

You are a senior AWS network architect. Produce a production-ready recommendation
choosing among Transit Gateway, VPC peering, and AWS PrivateLink for the environment
below, backed by explicit cost math and an IP-overlap safety check.

## Verify Reality First (do this before any recommendation)
1. Confirm the current pricing and quotas with the AWS Documentation MCP server
   (aws___read_documentation) — do NOT trust memorized prices. Confirm: TGW VPC
   attachment ($0.05/hr), TGW data processing ($0.02/GB), interface endpoint
   ($0.01/hr/AZ + $0.01/GB), VPC peering connection ($0), peering cross-AZ data
   ($0.01/GB each direction), same-AZ peering data ($0).
2. Confirm Transit Gateway, peering, and PrivateLink are available in the target region.
3. If I provided AWS access, run `aws ec2 describe-vpcs --query "Vpcs[].CidrBlock"` and
   `aws ec2 describe-vpc-peering-connections` to read REAL CIDRs and existing links.
   If not, state every assumption explicitly.
4. State the peering limit (125 active peerings per VPC) and TGW attachment quota
   (5000 attachments per gateway) and flag if my design approaches either.

## Inputs
- VPC count: [N]   Account layout: [single-account | multi-account/Organizations]
- Region: [us-east-1]   AZ placement of talking workloads: [same-AZ | cross-AZ]
- Traffic pattern: [full mesh | hub-and-spoke | one-to-many service access]
- Expected data per VPC pair: [GB/month]
- CIDR blocks per VPC: [list, or "discover via CLI"]

## Required Analysis
### 1. IP-Overlap Rules (hard gate — run first)
- Compare every CIDR pair. VPC peering and TGW routing BOTH break on overlapping CIDRs.
- If any overlap exists, you MUST recommend re-IP or PrivateLink (which tolerates overlap)
  and say so before cost. Non-overlapping is a prerequisite for peering/TGW — state it.

### 2. Decision Tree (by VPC count and direction)
- 2-3 VPCs, full mesh, no transitive need → VPC peering (connections free).
- 4+ VPCs needing any-to-any → peerings grow as N(N-1)/2; recommend Transit Gateway
  for routing simplicity once mesh management or the 125 limit bites.
- Hub-and-spoke / shared services across accounts → Transit Gateway.
- One-way access to a SINGLE service (not bidirectional routing), or overlapping CIDRs,
  or exposing to third parties → AWS PrivateLink interface endpoint.

### 3. Cost Crossover (compute the number)
- Build a table: monthly cost of peering vs TGW vs PrivateLink for MY GB/month and pair count.
- Peering: connection $0; same-AZ data $0/GB; cross-AZ $0.01/GB each way.
- TGW: attachments × $36/mo + total GB × $0.02/GB.
- PrivateLink: endpoints × $7.30/mo/AZ + GB × $0.01/GB.
- Solve for the GB/month where TGW total = peering total at MY topology. State it as
  "below X GB/month peering wins; above it, TGW's routing simplicity is worth $Y/mo."

## Deliverables
1. The recommended topology with one-paragraph justification tied to count + direction.
2. The CIDR-overlap report (pass/fail per pair).
3. The three-column monthly cost table and the computed crossover GB/month.
4. The decision tree showing why the alternatives were rejected.
5. Terraform (preferred) or AWS CLI scaffold for the chosen option, with placeholder
   CIDRs and tags Environment/Owner.

## Verification / Acceptance (emit evidence it works)
- Show the exact `aws___read_documentation` URLs you used to confirm each price.
- Provide a `terraform plan` expectation or `aws ec2 describe-*` command the user runs
  to confirm routes/attachments exist after applying.
- Provide a `aws ce get-cost-and-usage` filter to watch the data-processing line.

## Error Management
- If CIDRs overlap and the user still wants peering: STOP, explain routing will silently
  fail, recommend re-IP. This is a hard stop.
- If the user names PrivateLink for bidirectional routing: correct them.
- If region availability is unconfirmed: say so rather than assume.

Be imperative and specific. No preamble. Lead with the decision and the crossover number.
```

## What You Get

- A one-line topology decision (peering / Transit Gateway / PrivateLink) with justification.
- A CIDR-overlap report: pass/fail for every VPC pair, with re-IP guidance if any overlap.
- A three-column monthly cost table (peering vs TGW vs PrivateLink) computed for your GB/month.
- The exact **crossover GB/month** where Transit Gateway stops being cheaper than peering for your topology.
- A decision tree explaining why the two rejected options were rejected.
- A **Terraform** (or AWS CLI) scaffold for the chosen option, with placeholder CIDRs and `Environment`/`Owner` tags.
- A verification block: the documentation URLs used for pricing, a `describe-*` check, and a Cost Explorer filter to monitor the data-processing line.

## Example Output

Recommendation: VPC peering (full mesh). 3 VPCs, full mesh, same-AZ workloads, no transitive routing needed. CIDR overlap check: PASS (10.10.0.0/16, 10.20.0.0/16, 10.30.0.0/16 — disjoint). Monthly cost at 5 TB/month total: Peering $0 connections + $0 same-AZ data = $0/mo. Transit Gateway $108 (3 attachments) + $102.40 (5120 GB × $0.02) = $210.40/mo. PrivateLink: not applicable (bidirectional routing required). Crossover: peering wins at any volume while traffic stays same-AZ; even at full cross-AZ ($0.01/GB each way = $102.40/mo) peering stays below TGW until you exceed ~6 mesh pairs. Recommend Transit Gateway only once you reach 4+ VPCs needing any-to-any routing.

## AWS Services Used

Amazon VPC, AWS Transit Gateway, VPC peering, AWS PrivateLink (interface VPC endpoints), AWS Cost Explorer, AWS Resource Access Manager (cross-account TGW sharing), Amazon CloudWatch, AWS Identity and Access Management (IAM).

## Well-Architected Alignment

- **Cost Optimization**: Computes the explicit crossover GB/month and rejects the option that overpays; surfaces the $36/mo-per-attachment + $0.02/GB TGW tax versus $0 same-AZ peering before you commit.
- **Reliability**: The CIDR-overlap hard gate prevents silently broken routing; honors the 125-peering-per-VPC limit and the no-transitive-routing rule.
- **Operational Excellence**: Delivers an enforceable decision tree and a Terraform scaffold instead of a one-off click-through, plus a Cost Explorer filter for ongoing monitoring.
- **Security**: Steers overlapping-CIDR and third-party-exposure cases to PrivateLink (one-way, least-exposure access) rather than opening full network routes.
- **Performance Efficiency**: Keeps talking workloads same-AZ to avoid cross-AZ data charges and latency where the topology allows.

## Cost Notes

- VPC peering connections: **$0/mo**. Data: **$0/GB same-AZ**, **$0.01/GB each direction cross-AZ** in-region.
- Transit Gateway: **$0.05/hr per VPC attachment (~$36/mo)** + **$0.02/GB** data processing. Three attachments = **$108/mo** baseline.
- PrivateLink interface endpoint: **$0.01/hr per AZ (~$7.30/mo)** + **$0.01/GB** (first 1 PB/region).
- Worked example: 3-VPC mesh moving 5 TB/mo cross-AZ = peering **$102.40/mo** vs TGW **$210.40/mo**. Same-AZ = peering **$0** vs TGW **$210.40/mo**. The kit reports this so you do not default to TGW out of habit.

## Troubleshooting

- **AI recommends PrivateLink for general VPC-to-VPC connectivity** → Cause: traffic direction was omitted, so the model can't tell you need bidirectional routing. Fix: state "full mesh" or "hub routing" — PrivateLink is one-way access to a single service endpoint, not a routing fabric.
- **Routing silently fails after peering two VPCs** → Cause: overlapping CIDRs (e.g., both 10.0.0.0/16). Fix: re-IP one VPC to a disjoint block; peering and TGW both require non-overlapping CIDRs.
- **Pod A in VPC-A can't reach VPC-C through VPC-B's peering** → Cause: VPC peering is non-transitive. Fix: add a direct A-to-C peering, or move to Transit Gateway for hub routing.
- **Quoted prices don't match my bill** → Cause: prices vary by region and the model used a memorized figure. Fix: confirm the prompt's Verify-Reality step ran `aws___read_documentation` against the live pricing page for your region.
- **Hit "maximum peering connections" error** → Cause: 125 active peerings-per-VPC limit reached in a growing mesh. Fix: migrate to Transit Gateway (5000 attachments per gateway) — this is the crossover signal the decision tree flags.
