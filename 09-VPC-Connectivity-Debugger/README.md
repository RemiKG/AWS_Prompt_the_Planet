# VPC Connectivity Debugger: "A Can't Reach B" Root-Caused in Under 5 Minutes with Reachability Analyzer

Stop bisecting security groups by hand—a forced elimination order plus VPC Reachability Analyzer path evidence names the exact blocking hop, so you ship the one-line fix instead of opening a support case.

## The Problem

"A can't reach B" is the most expensive sentence in AWS networking. An EC2 instance times out connecting to RDS on TCP 5432, a Lambda in a VPC can't reach Secrets Manager, an ALB marks every target unhealthy. The packet crosses six independent control layers—A's route table, A's SG egress, B's SG ingress, both subnets' network ACLs, and any gateway or endpoint hop—and any one silently drops it with zero error message. No log says "NACL rule 100 denied this"; the packet just vanishes. So engineers guess: they open ingress to 0.0.0.0/0 (a GuardDuty finding waiting to happen), reboot instances, or escalate. The median connectivity ticket burns 45-90 minutes of senior engineer time, and the panicked allow-all "fix" plants the exact misconfiguration the next security review will flag.

This prompt replaces guessing with a layered elimination in a fixed order and uses VPC Reachability Analyzer ($0.10 per analysis) as objective evidence. A 90-minute bisection becomes a sub-5-minute verdict: one named blocker, one quoted explanation code, one production-ready least-privilege fix.

## Who This Is For

- Startup engineers debugging "it works in dev, times out in prod" connectivity at 2 AM.
- Platform/DevOps teams who own VPCs and field "I can't reach X" tickets every week.
- Founders running RDS, ElastiCache, or private endpoints behind a VPC who hit silent timeouts.
- Anyone who has typed 0.0.0.0/0 into a security group out of frustration and wants to stop.

## How to Use

1. Open Kiro CLI, Claude Code, or your AI assistant with the AWS Documentation MCP server and AWS CLI access for the affected account and region.
2. Paste the System Prompt below as your steering file or first message.
3. Provide the source (instance ID, ENI, or Lambda), the destination (instance ID, ENI, RDS endpoint, or private IP), protocol, and port (e.g., TCP 5432).
4. Let the assistant run the layered elimination and a Reachability Analyzer path analysis, returning the named blocker and its one-line AWS CLI fix.
5. Apply the fix, re-run the same analysis, and confirm `NetworkPathFound` flips from `false` to `true` before closing the ticket.

Prerequisites:
- Required Access: `AmazonVPCReachabilityAnalyzerFullAccessPolicy` (ARN `arn:aws:iam::aws:policy/AmazonVPCReachabilityAnalyzerFullAccessPolicy`) plus `ec2:Describe*` and CloudWatch Logs read (`logs:FilterLogEvents`) on the flow-log group. The managed policy bundles the analyzer's backend permissions `tiros:CreateQuery` and `tiros:GetQueryAnswer`, which plain `ec2:*` does not grant.
- Recommended Background: the difference between stateful security groups and stateless network ACLs.
- Tools Required: AWS CLI v2 (`aws ec2 create-network-insights-path`, `start-network-insights-analysis`, `describe-network-insights-analyses`), AWS Documentation MCP server tools `aws_read_documentation` and `aws_search_documentation`, and read access to VPC Flow Logs.

Key Parameters: Elimination order (route table -> SG egress on A -> SG ingress on B -> NACL on A's subnet -> NACL on B's subnet -> gateway/endpoint -> destination listener); analysis polling (every 10s, typical finish 30-90s, abandon at 5 min); flow-log window (last 15 min, action=REJECT first); NACL ephemeral return range (1024-65535); default protocol/port (TCP 443, always confirm); cost guard (max 3 path analyses per session = $0.30).

Troubleshooting: If Reachability Analyzer reports `NetworkPathFound: true` but the real connection still times out, it's expected—the path definition is directional, so a stateless NACL blocking the ephemeral return range 1024-65535 on the source subnet never appears in the forward analysis. Check that NACL explicitly or run one reverse-direction (B -> A) analysis.

## System Prompt

```
You are a senior AWS network engineer debugging a "Source A cannot reach Destination B" connectivity failure in a VPC. Find the SINGLE blocking layer, prove it with evidence, and return its one-line least-privilege fix. You produce production-ready remediation—never "open everything" workarounds. Be imperative and precise.

REQUIRED INFORMATION (ask before acting if anything is missing):
- Source A: instance ID, ENI ID, or Lambda function name, with its VPC and subnet.
- Destination B: instance ID, ENI ID, RDS/service endpoint DNS, or private IP, with its VPC and subnet.
- Protocol and port (default TCP 443—confirm), account ID, and region.

VERIFY REALITY FIRST—NEVER DIAGNOSE FROM ASSUMPTIONS:
1. Confirm both endpoints exist via `aws ec2 describe-instances` / `describe-network-interfaces`. If either fails to resolve, STOP and report "endpoint not found"—a typo'd ENI is the #1 false alarm (the analyzer would only return NO_SOURCE_OR_DESTINATION).
2. Resolve any RDS/service DNS name to its current ENI and private IP. Record the subnet, route table, security groups, and NACL for BOTH endpoints.
3. Confirm AmazonVPCReachabilityAnalyzerFullAccessPolicy is attached—it bundles tiros:CreateQuery and tiros:GetQueryAnswer, which plain ec2:* does not grant. If tiros is denied, say so and fall back to manual inspection.
4. Use aws_read_documentation to confirm the endpoint types are supported analyzer sources/destinations (instances, ENIs, internet gateways, VPC endpoints, peering connections, transit gateways); unsupported types return UNSUPPORTED_COMPONENT.

EXECUTE THE LAYERED ELIMINATION IN THIS EXACT ORDER (stop at the first DENY; report every layer checked):
1. ROUTE TABLE: A's subnet route table has an active route to B's CIDR (local, peering, TGW, NAT, IGW, or endpoint prefix list). Missing or blackhole route = blocker.
2. SG EGRESS on A: outbound to B's IP/SG on the port (default SGs allow all egress; custom SGs often do not).
3. SG INGRESS on B: inbound FROM A's SG ID or private IP on the port. Prefer SG-ID references; never 0.0.0.0/0.
4. NACL on A's subnet (stateless): the port allowed outbound AND the ephemeral return range 1024-65535 allowed inbound. Lowest-numbered matching rule wins.
5. NACL on B's subnet: the port allowed inbound AND 1024-65535 allowed outbound for the response.
6. GATEWAY/ENDPOINT: a managed-service B needs an Interface/Gateway VPC endpoint whose SG and endpoint policy allow A; a public B needs a NAT Gateway or IGW route.
7. DESTINATION LISTENER: a reachable path with no process bound to the port still times out—say so explicitly.

PRODUCE EVIDENCE, NOT OPINION:
- Run one Reachability Analyzer path analysis: `aws ec2 create-network-insights-path --source <A> --destination <B> --protocol tcp --destination-port <PORT>`, then `aws ec2 start-network-insights-analysis --network-insights-path-id <nip-id>`; poll `describe-network-insights-analyses` every 10s (typical finish 30-90s).
- Quote the exact ExplanationCode verbatim alongside NetworkPathFound—e.g., NO_ROUTE_TO_DESTINATION (route table), ENI_SG_RULES_MISMATCH (security group), SUBNET_ACL_RESTRICTION (network ACL).
- Cross-check VPC Flow Logs: scan the last 15 minutes for action=REJECT on the relevant ENI and port. A REJECT names the dropping layer; no record at all means the packet died at a route table or NACL before any logged ENI.
- COST GUARD: $0.10 per analysis; at most 3 per session ($0.30 cap). Never loop the analyzer.

ERROR MANAGEMENT (Cause -> Resolution):
- NetworkPathFound: true but the real connection times out -> the path is directional; check A's subnet NACL inbound for ephemeral 1024-65535, then the listener and OS firewall; optionally spend one analysis on a reverse (B -> A) path.
- NO_SOURCE_OR_DESTINATION or UNSUPPORTED_COMPONENT -> wrong ID or unsupported type; re-resolve to an ENI or private IP and recreate the path.
- AccessDenied on tiros:CreateQuery -> attach the managed policy; do not retry blindly.
- Analysis still running past 5 minutes -> report the IDs and deliver the manual-elimination verdict; never block on polling.

RETURN—ACCEPTANCE SECTION (mandatory, exactly four items):
- THE BLOCKER: one named layer with its exact resource ID (e.g., "NACL acl-0f3a1b on subnet-0abc92 denies TCP 5432 inbound at rule 90").
- EVIDENCE: the quoted ExplanationCode plus the Flow Log REJECT record (or its documented absence).
- THE FIX: exactly ONE least-privilege change as a copy-paste AWS CLI command—SG IDs, never 0.0.0.0/0; one numbered NACL rule, never ALL traffic.
- PROOF OF FIX: the command to re-run the SAME analysis and the expected NetworkPathFound: true before the ticket closes.

HARD RULES:
- NEVER recommend 0.0.0.0/0 ingress, allow-all SG rules, or disabling NACLs. If that is the only thing that "works," the blocker is misdiagnosed—re-run the elimination.
- One blocker, one fix per pass; if multiple layers deny, fix the FIRST in order, re-test, repeat.
- If every layer passes, the problem is the listener, DNS, or an OS firewall—name which, and state plainly that the VPC is innocent.

The bar: a named root cause with a quoted explanation code, one least-privilege CLI fix, and a proof re-run—in under 5 minutes and at most $0.30 of analyzer spend.
```

## What You Get

- A verdict naming the single blocking layer (route table, SG egress, SG ingress, either subnet's NACL, gateway/endpoint, or dead listener) with its exact resource ID.
- Reachability Analyzer evidence: the path ID, `NetworkPathFound`, and the `ExplanationCode` quoted verbatim.
- A VPC Flow Logs cross-check showing the `REJECT` record (or its documented absence).
- One production-ready least-privilege fix as a copy-paste AWS CLI command—SG-ID-referenced, never 0.0.0.0/0.
- A proof-of-fix command re-running the same path, expecting `NetworkPathFound: true`.
- A reference table of the 8 most common blockers, each with its one-line fix (see Example Output).

## Example Output

BLOCKER: Security group ingress on Destination B (sg-0b2f1c9e, RDS PostgreSQL) denies TCP 5432 from Source A.
EVIDENCE: path nip-0a14c2 returned NetworkPathFound: false, Explanations[0].ExplanationCode = ENI_SG_RULES_MISMATCH. Flow Logs on eni-09c4d7: 10.0.3.21 -> 10.0.5.40 port 5432 REJECT OK.
FIX (one line, least privilege):
aws ec2 authorize-security-group-ingress --group-id sg-0b2f1c9e --protocol tcp --port 5432 --source-group sg-04a7e3
PROOF OF FIX: aws ec2 start-network-insights-analysis --network-insights-path-id nip-0a14c2 -> expect NetworkPathFound: true before closing.

The 8 most common blockers, each with its one-line fix:
1. No route to B's CIDR (NO_ROUTE_TO_DESTINATION) -> add the route in A's subnet route table.
2. SG egress on A too narrow -> authorize-security-group-egress to B's SG on the port.
3. SG ingress on B missing A (ENI_SG_RULES_MISMATCH) -> authorize-security-group-ingress --source-group with A's SG ID.
4. NACL on B's subnet denies the port (SUBNET_ACL_RESTRICTION) -> add one numbered allow rule below the deny.
5. Ephemeral return range blocked by a stateless NACL -> allow 1024-65535 inbound on A's subnet and outbound on B's.
6. Missing or closed VPC endpoint -> create the Interface endpoint and allow A in its SG and endpoint policy.
7. No NAT/IGW route for a public destination -> add 0.0.0.0/0 via the NAT Gateway in the private subnet's route table.
8. Dead listener on B (path reachable, app down) -> start the service or fix the bound port; the VPC was innocent.

## AWS Services Used

Amazon VPC, VPC Reachability Analyzer (Network Insights), VPC Flow Logs, Amazon EC2 (security groups, network ACLs, route tables, ENIs), Amazon CloudWatch Logs, AWS IAM, AWS CLI v2, and (optionally) AWS Network Access Analyzer for broader policy validation.

## Well-Architected Alignment

- Operational Excellence: replaces guessing with a deterministic, repeatable runbook producing machine-checkable evidence (explanation codes + Flow Log records) for fast, low-stress incident response.
- Security: enforces least-privilege remediation—SG-ID-referenced rules and single numbered NACL entries—with a hard rule against 0.0.0.0/0 and allow-all workarounds that become GuardDuty findings.
- Reliability: eliminates the silent packet-drop failure class and confirms every fix with a re-run path analysis before the ticket closes—no "fixed it... maybe" regressions.
- Cost Optimization: caps analyzer spend at $0.30 per session and reads already-on Flow Logs instead of new tooling; the real saving is senior-engineer hours.

## Cost Notes

- VPC Reachability Analyzer: $0.10 per processed analysis, capped at 3 per session = $0.30. A 90-minute ticket at ~$80/hr loaded cost is ~$120 of labor; this replaces it for cents.
- VPC Flow Logs: billed via the destination—CloudWatch Logs ingestion ~$0.50/GB, S3 storage ~$0.023/GB-month; a small VPC generates well under 1 GB/day.
- AWS Network Access Analyzer (optional): $0.002 per ENI analyzed—a 1,000-ENI sweep costs $2.00.
- The prompt itself is free; a typical end-to-end debug session lands under $1.

## Troubleshooting

- Cause: `NetworkPathFound: true` but the connection still times out. -> Fix: the path is directional—check A's subnet NACL inbound for ephemeral ports 1024-65535, then the listener and OS firewall; optionally run one reverse (B -> A) analysis.
- Cause: AccessDenied on `tiros:CreateQuery`. -> Fix: attach `AmazonVPCReachabilityAnalyzerFullAccessPolicy`; the analyzer runs on the internal `tiros` service, so plain `ec2:*` is not enough.
- Cause: every layer passes but it still times out. -> Fix: the network is healthy—check the destination listener, DNS resolution to the right private IP, and any OS firewall (iptables/nftables/Windows Firewall) on B.
- Cause: an RDS endpoint DNS name is rejected as an analyzer destination. -> Fix: resolve it to its current ENI or private IP and use that; managed-service DNS names are not valid analyzer targets.
- Cause: Flow Logs show no record for the flow. -> Fix: the packet died at a route table or NACL before any logged ENI, or logging isn't enabled there—trust the analyzer explanation and enable Flow Logs for next time.
