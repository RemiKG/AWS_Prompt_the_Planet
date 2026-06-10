# AWS Prompt the Planet submissions

30 entries for the AWS Prompt the Planet challenge (DoraHacks, June 2026). One
folder per entry, content as submitted. Each is a system prompt for an agentic
assistant with AWS access: Kiro CLI, Claude Code, Amazon Q Developer CLI,
anything that can run the AWS CLI.

These were written with heavy AI assistance, which the challenge allows. My
process per package: pick a problem I'd hit or seen someone hit, write a spec
for what the prompt must do and must refuse to do, generate drafts with an AI
assistant, then check the load-bearing claims against AWS docs (action names,
CLI syntax, quotas, prices, version behavior like pg_stat_statements renaming
total_time to total_exec_time in PG13). Claims I couldn't verify got cut or
rewritten as "confirm this in the console."

I drafted more than I could enter and submitted the 30 I got through before
the deadline closed. Hence the gaps in the numbering (06, 22, 29) and the
three folders numbered 61-63, which came from a later pass. Folder names match
the submitted links so I'm not renaming anything, including 27, which says
Redis in the path but targets ElastiCache for Valkey.

Caveats:

- Prices and quotas were checked in June 2026 and will drift.
- "Example Output" sections show the expected shape of a response. They are
  illustrative, not captured transcripts. I verified the individual commands,
  queries, and API fields, but I have not run all 30 prompts end to end
  against live accounts.
- The logo.png in each folder is script-generated decoration.

All 30 follow the same rules: verify before recommending (confirm the
resource, version, or quota actually exists, and say "not verified" instead of
guessing), read-only for diagnosis (anything mutating needs explicit human
approval), and end with a verification step that defines done.

Usage: copy the fenced block under "System Prompt" into your assistant's
system prompt or steering file (.kiro/steering/ for Kiro, CLAUDE.md for Claude
Code), fill in the [BRACKETED] placeholders, and use credentials with the
access level the package's Prerequisites section lists. Usually read-only.

## The packages

### Debugging and incident response

| Folder | What it does |
|---|---|
| [01-IAM-Access-Denied-Debugger](01-IAM-Access-Denied-Debugger/) | Traces an AccessDenied to the decisive one of the five policy types using CloudTrail and the policy simulator, then writes the minimal scoped fix. Refuses wildcard grants. |
| [07-Terraform-State-Surgery](07-Terraform-State-Surgery/) | Import, move, split, and drift remediation for S3-backed Terraform state, with a backup first and a dry run before every operation. |
| [09-VPC-Connectivity-Debugger](09-VPC-Connectivity-Debugger/) | Root-causes "A cannot reach B" by layered elimination plus a Reachability Analyzer path trace. |
| [28-RDS-Postgres-Performance-Triage](28-RDS-Postgres-Performance-Triage/) | Maps the top Performance Insights wait event to specific diagnostic queries, then splits fixes into safe-now versus needs-a-maintenance-window. |

### Cost reduction

| Folder | What it does |
|---|---|
| [04-NAT-Gateway-Egress-Cost](04-NAT-Gateway-Egress-Cost/) | Attributes NAT gateway traffic by destination service and replaces it with VPC endpoints where the math works. |
| [08-Lambda-Cold-Start-Tuning-Kit](08-Lambda-Cold-Start-Tuning-Kit/) | Runs Lambda Power Tuning, builds a SnapStart eligibility table, and shows the provisioned-concurrency break-even arithmetic. |
| [10-S3-Lifecycle-Optimizer](10-S3-Lifecycle-Optimizer/) | Turns S3 Inventory data into storage-class decisions that account for retrieval and transition fees. |
| [11-Savings-Plans-RI-Analyzer](11-Savings-Plans-RI-Analyzer/) | Builds a laddered commitment plan from Cost Explorer, CUR, and Compute Optimizer data. |
| [24-Bedrock-Batch-Inference-Half-Price](24-Bedrock-Batch-Inference-Half-Price/) | Converts bulk LLM workloads to Bedrock batch inference jobs (JSONL, IAM role, CLI call) at 50 percent of on-demand pricing. |
| [30-Multi-VPC-Topology-Decision-Kit](30-Multi-VPC-Topology-Decision-Kit/) | Cost-crossover analysis for Transit Gateway versus VPC peering versus PrivateLink. |

### Identity and auth

| Folder | What it does |
|---|---|
| [16-IAM-Identity-Center-SSO-Migration](16-IAM-Identity-Center-SSO-Migration/) | Migrates IAM users to Identity Center with permission-set parity checks and a break-glass account kept outside SSO. |
| [17-Cognito-Production-Auth-Setup](17-Cognito-Production-Auth-Setup/) | Cognito user pool with hosted UI on a custom domain, OAuth federation, and MFA. |
| [62-IAM-Least-Privilege-Katas](62-IAM-Least-Privilege-Katas/) | Five graded policy-repair drills in a sandbox account, scored only by simulate-principal-policy. |

### CI/CD and ephemeral environments

| Folder | What it does |
|---|---|
| [05-Keyless-CICD-OIDC-Deploy](05-Keyless-CICD-OIDC-Deploy/) | GitHub Actions to AWS via OIDC trust roles scoped to one repo and branch. No long-lived keys. |
| [13-Ephemeral-PR-Preview-Environments](13-Ephemeral-PR-Preview-Environments/) | A full-stack preview environment per pull request, with teardown guaranteed by PR-close hooks plus a nightly reaper. |

### Databases and caching

| Folder | What it does |
|---|---|
| [02-Aurora-Postgres-Production-Kit](02-Aurora-Postgres-Production-Kit/) | Aurora PostgreSQL sizing, RDS Proxy connection pooling, and a restore actually executed against a scratch instance. |
| [03-DynamoDB-Single-Table-Design](03-DynamoDB-Single-Table-Design/) | Single-table DynamoDB modeling driven by an access-pattern worksheet instead of an entity diagram. |
| [27-ElastiCache-Redis-Production-Kit](27-ElastiCache-Redis-Production-Kit/) | ElastiCache for Valkey: cluster-mode decision table, a failover drill you actually trigger, and stampede-resistant caching code. |

### Data processing and pipelines

| Folder | What it does |
|---|---|
| [18-Step-Functions-Distributed-Map](18-Step-Functions-Distributed-Map/) | Fans a million S3 objects across parallel Lambdas with batching and failure thresholds. |
| [19-AWS-Batch-Spot-Checkpoint](19-AWS-Batch-Spot-Checkpoint/) | AWS Batch on EC2 Spot with a 2-minute interruption handler, S3 checkpoints, and auto-resume. |
| [20-Kinesis-Iceberg-Athena-Clickstream](20-Kinesis-Iceberg-Athena-Clickstream/) | Clickstream pipeline: Kinesis to Firehose to S3 Iceberg tables queried in Athena. |
| [21-Glue-Athena-Iceberg-Lakehouse](21-Glue-Athena-Iceberg-Lakehouse/) | Glue, Athena, and Iceberg lakehouse with partitioning, compaction, and per-query byte caps. |
| [23-Serverless-Document-Extraction-Pipeline](23-Serverless-Document-Extraction-Pipeline/) | Textract plus Bedrock structured extraction, with low-confidence fields routed to A2I human review. |

### DNS, certificates, and CDN

| Folder | What it does |
|---|---|
| [14-Route53-ACM-DNS-Kit](14-Route53-ACM-DNS-Kit/) | Route 53 and ACM in Terraform, including DNS validation and the us-east-1 certificate requirement for CloudFront. |
| [15-CloudFront-Production-CDN-Kit](15-CloudFront-Production-CDN-Kit/) | CloudFront with origin access control, signed URLs, and origin failover. |

### Migrations and upgrades

| Folder | What it does |
|---|---|
| [12-Graviton-Migration-Runbook](12-Graviton-Migration-Runbook/) | Per-runtime Graviton migration (ECS, EKS, Lambda, RDS) with multi-arch builds and tested rollbacks. |
| [25-Zero-Drama-EKS-Upgrades](25-Zero-Drama-EKS-Upgrades/) | EKS minor-version upgrade runbook with version-skew checks, PDB audits, and surge settings. |
| [26-Heroku-to-AWS-Cutover-Kit](26-Heroku-to-AWS-Cutover-Kit/) | Heroku to AWS: Procfile and config-var translation, DMS-validated database move, weighted Route 53 cutover with rollback. |

### Learning an account

| Folder | What it does |
|---|---|
| [61-AWS-Account-Tour-Guide](61-AWS-Account-Tour-Guide/) | Read-only guided tour of a real AWS account for new engineers, ending in a quiz built from the account's actual resources. |
| [63-Five-Dollar-Cert-Lab](63-Five-Dollar-Cert-Lab/) | SAA-C03 exam domains mapped to hands-on labs on a real account, under five dollars total, with teardown proven by CLI after each lab. |

## License

MIT, see LICENSE.

Kenneth (GitHub: RemiKG)
