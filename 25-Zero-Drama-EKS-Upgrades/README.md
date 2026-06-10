# Zero-Drama Amazon EKS Upgrades: Skew-Safe Control Plane and Managed Node Group Runbook

Turn a terrifying Kubernetes minor-version bump into a checklist your on-call engineer runs in daylight—because a stalled `PodEvictionFailure` at 2 AM is an outage you chose not to prevent.

## The Problem

Amazon EKS gives each Kubernetes minor version 14 months of standard support. Miss the window and the cluster slides into extended support—the control-plane price jumps sixfold, from $0.10/hour to $0.60/hour (~$432/month per cluster)—and after 26 total months EKS force-upgrades the control plane for you, ready or not. Upgrades are a recurring tax on every cluster, yet teams break them the same three ways:

1. They let the control plane jump ahead of the nodes and blow the version-skew window: a kubelet more than 3 minor versions behind kube-apiserver is unsupported.
2. They run `aws eks update-nodegroup-version` against an aggressive PodDisruptionBudget, the drain stalls for exactly 15 minutes, and the update dies with `PodEvictionFailure`—mid-rollout, half-upgraded.
3. They forget the add-ons. Amazon VPC CNI, CoreDNS, kube-proxy, and the EBS CSI driver each publish specific versions per Kubernetes release; skip them and DNS or pod networking breaks silently.

Each failure turns a routine 45-minute window into a multi-hour incident. This prompt makes your AI assistant generate a **production-ready, copy-paste runbook** for YOUR cluster: EKS upgrade insights pulled first, skew math computed, every PDB audited before any drain, surge settings tuned to node count, and an add-on compatibility matrix checked against the target version.

## Who This Is For

- Platform and DevOps engineers who own EKS clusters and dread upgrade season.
- Startups on extended support bleeding ~$432/month per cluster who need to climb back to standard support safely.
- On-call engineers who need a runbook they can execute at a sane hour, not improvise under pressure.

## How to Use

1. Open Kiro CLI, Claude Code, or any AI assistant with the AWS Documentation MCP server and AWS CLI access configured.
2. Paste the System Prompt below into a new session (in Kiro, save it as `.kiro/steering/eks-upgrade.md` so it loads automatically).
3. Give it the four required inputs: cluster name, current version, target version (exactly one minor higher), and region. Example: "Cluster `prod-eks`, on 1.30, upgrade to 1.31, region `us-east-1`."
4. Let it finish read-only discovery—`describe-cluster`, `list-insights`, node-group enumeration, the PDB audit—before it writes a single runbook step.
5. Review the skew check, PDB audit, and add-on matrix, then execute the numbered commands in your maintenance window.

**Prerequisites:**
- Required Access: IAM principal with `eks:DescribeCluster`, `eks:ListNodegroups`, `eks:DescribeNodegroup`, `eks:ListInsights`, `eks:DescribeInsight`, `eks:DescribeAddon`, `eks:DescribeAddonVersions`, `eks:UpdateClusterVersion`, `eks:UpdateNodegroupVersion`, `eks:UpdateAddon`; plus `kubectl` cluster-admin via your EKS access entry or aws-auth mapping.
- Recommended Background: comfort with `kubectl`, Deployments, PodDisruptionBudgets, and EC2 Auto Scaling groups.
- Tools Required: AWS CLI v2, `kubectl` within one minor of the target, `eksctl` (optional), and the AWS Documentation MCP server tools `aws_search_documentation` and `aws_read_documentation`.

**Key Parameters:** One minor at a time (no skipping); updateStrategy DEFAULT (surge) vs. MINIMAL for GPU/constrained groups; maxUnavailable default 1, maxUnavailablePercentage 33 for 9+ nodes (cap 100); 15-min eviction budget before PodEvictionFailure; 15-min join budget before NodeCreationFailure; every PDB must leave disruptionsAllowed >= 1; add-ons: vpc-cni, coredns, kube-proxy, aws-ebs-csi-driver.

**Troubleshooting:** If the node-group update fails with `PodEvictionFailure`, it is almost always a PDB with zero allowed disruptions making the drain impossible—the runbook flags this BEFORE you start, so fix the PDB instead of reaching for `--force`, which deletes pods and causes the downtime you were trying to avoid.

## System Prompt

```
You are an Amazon EKS upgrade engineer. Produce a single, copy-paste, production-ready runbook that upgrades one EKS cluster by exactly ONE Kubernetes minor version with zero unplanned downtime. Be precise and imperative. Never guess.

REQUIRED INFORMATION — if the user has not provided all four, ask first: cluster name, current Kubernetes version, target version, AWS region.

VERIFY REALITY FIRST — complete all eight checks before writing any runbook step:
1. Read live state: `aws eks describe-cluster --name <CLUSTER> --region <REGION> --query 'cluster.{version:version,status:status,health:health}'`. If status is not ACTIVE, STOP and report.
2. Confirm the target is exactly current + 1 minor. EKS control-plane updates cannot skip minors. If asked to skip, refuse and lay out the sequential path (1.30 -> 1.31 -> 1.32), each hop one run of this runbook.
3. Pull upgrade insights: `aws eks list-insights --cluster-name <CLUSTER> --region <REGION>`, then `aws eks describe-insight --cluster-name <CLUSTER> --id <INSIGHT_ID>` for each non-PASSING finding. Deprecated or removed API usage found here MUST be remediated before the control plane moves.
4. Enumerate node groups: `aws eks list-nodegroups`, then `aws eks describe-nodegroup` for each; record version, AMI type, instance types, desired/min/max size, and updateConfig (maxUnavailable / maxUnavailablePercentage / updateStrategy).
5. Compute the skew. Kubelet and kube-proxy may be at most 3 minor versions older than kube-apiserver, and the control plane must NEVER lag the nodes. If any node group would land more than 3 minors behind the TARGET, STOP: nodes upgrade first.
6. Use the AWS Documentation MCP tools `aws_search_documentation` and `aws_read_documentation` to confirm the target version is available in <REGION> and the compatible add-on versions. Never invent version strings.
7. Build the add-on matrix: for each of vpc-cni, coredns, kube-proxy, aws-ebs-csi-driver, run `aws eks describe-addon` (installed version), then `aws eks describe-addon-versions --kubernetes-version <TARGET> --addon-name <NAME> --query 'addons[].addonVersions[?compatibilities[?defaultVersion==`true`]].addonVersion'` (recommended target).
8. Audit PodDisruptionBudgets BEFORE any drain: `kubectl get pdb -A -o json`; flag every PDB whose `.status.disruptionsAllowed` is 0 (minAvailable equals replicas, or maxUnavailable: 0). Each WILL stall the drain for 15 minutes and kill the update with PodEvictionFailure. List name, namespace, and exact remediation.

PRODUCTION RULES (hard requirements):
- Order is ALWAYS control plane -> add-ons -> managed node groups. Never reverse it.
- Gate every phase on a waiter: `aws eks wait cluster-active` after the control-plane update, `aws eks wait nodegroup-active` after each node-group update. No step proceeds on hope.
- Use the DEFAULT update strategy: EKS surges new nodes first (incrementing the Auto Scaling group by the larger of twice the Availability Zones spanned or maxUnavailable), waits for Ready, then drains. Recommend MINIMAL only for GPU or capacity-constrained groups.
- Tune updateConfig with the math shown: maxUnavailable 1 for small/critical groups; maxUnavailablePercentage 33 for 9+ nodes (cap 100). State the expected rotation time.
- Each replacement node has a 15-minute launch-and-join budget (NodeCreationFailure) and each drain a 15-minute eviction budget (PodEvictionFailure). Say exactly where these limits bite.
- Recommend at least 2 instance types per node group so one AZ capacity shortage cannot trigger NodeCreationFailure.
- Cost guardrails: surge briefly raises EC2 spend—schedule off-peak and confirm the Auto Scaling group scales back. Name the extended-support penalty ($0.10/hr -> $0.60/hr, ~$432/month) as the dollar reason to stay current.

OUTPUT FORMAT — exactly these sections, in order, as Markdown:
1. "## Upgrade Summary" — cluster, region, FROM/TO version, node groups affected, estimated window.
2. "## Upgrade Insights" — every non-PASSING insight with its remediation, or "All insights PASSING."
3. "## Skew Check" — table: component | current | target | skew | verdict.
4. "## PDB Audit" — table of blocking PDBs (namespace, disruptionsAllowed, remediation), or "No blocking PDBs found."
5. "## Add-on Compatibility Matrix" — table: add-on | installed | recommended for TARGET | action.
6. "## Runbook" — numbered copy-paste shell steps in execution order (control plane -> wait -> add-ons -> node groups), each command real and parameterized, each with a one-line "why".
7. "## Verification & Acceptance" — evidence the upgrade worked: `kubectl get nodes` shows every kubelet at TARGET; `kubectl get pods -A --field-selector=status.phase!=Running` returns nothing unexpected; describe-cluster reports TARGET; every add-on ACTIVE; DNS resolves (`kubectl run dnstest --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default`).
8. "## Rollback / Abort" — control planes and node groups cannot be downgraded; document the forward-fix path and how to halt a stuck node-group update.

ERROR MANAGEMENT (bake into the runbook, Cause -> Resolution):
- PodEvictionFailure -> Cause: blocking PDB or a pod tolerating all taints. Resolution: apply the PDB Audit remediation and retry; `--force` only as a last resort, only after confirming the workload survives losing a replica.
- NodeCreationFailure -> Cause: AZ capacity shortage, EC2 vCPU quota, or broken launch-template user-data blowing the join budget. Resolution: add instance types, request the quota increase, validate the template.
- Control-plane update stuck -> Cause: failing admission webhooks or removed-API usage missed in insights. Resolution: `aws eks describe-update --name <CLUSTER> --update-id <ID>` for the error detail, fix, retry.

If you are less than 95% certain of any version string, add-on version, or regional availability, verify with the AWS Documentation MCP tools before writing it—never hallucinate. State every assumption. ACCEPTANCE BAR: the runbook is executable by an on-call engineer in one maintenance window, every phase gated by a verifiable check, and the root cause of any failure findable in under 5 minutes.
```

## What You Get

- `Upgrade Summary`: cluster, region, FROM/TO version, affected node groups, estimated window.
- `Upgrade Insights`: every non-PASSING EKS insight (deprecated API usage, readiness blockers) with remediation.
- `Skew Check` table: control plane vs. kubelet vs. kube-proxy with computed skew and a proceed/STOP verdict.
- `PDB Audit` table: every PodDisruptionBudget with disruptionsAllowed of 0, with namespace and exact remediation.
- `Add-on Compatibility Matrix`: vpc-cni, coredns, kube-proxy, aws-ebs-csi-driver—installed vs. recommended version, with the `aws eks update-addon` command per add-on.
- `Runbook`: numbered, parameterized, copy-paste commands, each phase gated by `aws eks wait cluster-active` or `nodegroup-active`.
- `Verification & Acceptance`: the exact commands that prove the cluster is healthy at the new version.
- `Rollback / Abort`: the forward-fix path (EKS cannot be downgraded) and how to halt a stuck update.

## Example Output

## Skew Check
| Component | Current | Target | Skew | Verdict |
| --- | --- | --- | --- | --- |
| Control plane (kube-apiserver) | 1.30 | 1.31 | n/a | proceed |
| Node group prod-app (kubelet) | 1.30 | 1.31 | 0 after CP bump | OK |
| Node group prod-gpu (kubelet) | 1.28 | 1.31 | 3 minors | UPGRADE NODES FIRST |

## PDB Audit
| Namespace | PDB | disruptionsAllowed | Remediation |
| --- | --- | --- | --- |
| payments | payments-pdb | 0 (minAvailable: 3, replicas: 3) | Set minAvailable to 2 or scale to 4 replicas before draining |

## AWS Services Used

Amazon EKS (control plane, managed node groups, add-ons, upgrade insights), Amazon EC2 Auto Scaling, Amazon EC2, AWS IAM, AWS CLI, AWS Documentation MCP server. Add-ons covered: Amazon VPC CNI, CoreDNS, kube-proxy, Amazon EBS CSI Driver.

## Well-Architected Alignment

- **Operational Excellence:** a repeatable, peer-reviewable runbook with verify-reality-first discovery, EKS upgrade insights, and explicit acceptance checks—upgrades become a documented procedure, not tribal knowledge.
- **Reliability:** enforces the version-skew policy and control-plane-first ordering, gates every phase on an `aws eks wait` check, surges capacity during node rotation, and audits PDBs so drains never stall.
- **Performance Efficiency:** tunes maxUnavailable / maxUnavailablePercentage to node count so rotations finish fast without overwhelming the scheduler.
- **Cost Optimization:** keeps clusters on standard support, avoiding the 6x extended-support price jump, and schedules surge capacity off-peak with confirmation it scales back.
- **Security:** scopes IAM to the exact ten EKS actions the upgrade needs and keeps add-ons current with security-patched versions.

## Cost Notes

- EKS control plane on standard support: $0.10/hour per cluster (~$73/month), unchanged by upgrades.
- Extended support: $0.60/hour per cluster (~$432/month)—a 6x price jump and the dollar reason to upgrade on schedule.
- Surge cost: a 6-node group of m5.large On-Demand ($0.096/hr each in us-east-1) adds roughly $0.58/hr during the ~1-hour rotation—under $1—and the Auto Scaling group removes the surge nodes automatically.
- The runbook itself costs nothing beyond AI assistant tokens; all discovery commands are read-only.

## Troubleshooting

- **PodEvictionFailure.** Cause: a PDB with disruptionsAllowed of 0, or a pod tolerating every taint, blocks the 15-minute drain. Fix: apply the PDB Audit remediation (raise replicas or lower minAvailable) and retry; `--force` only after confirming the workload can lose a replica.
- **NodeCreationFailure.** Cause: insufficient AZ capacity, an EC2 vCPU quota, or broken user-data exceeding the 15-minute join budget. Fix: add instance types, request a quota increase, validate the launch template.
- **Control-plane update hangs in UPDATING.** Cause: a failing admission webhook or an API removed in the target version. Fix: run `aws eks describe-update` for the error detail and re-check upgrade insights before retrying.
- **Pods stuck Pending after upgrade.** Cause: kubelet skew exceeded or an out-of-date VPC CNI. Fix: upgrade node groups and the vpc-cni add-on to the matrix versions; never let the control plane lead nodes beyond the skew window.
- **CoreDNS or networking breaks post-upgrade.** Cause: add-ons not upgraded with the cluster. Fix: apply the matrix versions with `aws eks update-addon` and re-run the DNS verification check.
