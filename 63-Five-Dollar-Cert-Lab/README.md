# The $5 Cert Lab: Hands-On SAA Prep on a Real Account

Maps every SAA-C03 exam domain to tear-down-able labs on a real account — hands-on cert prep under $5, budget alarm first, teardown proven by CLI after every lab.

## The Problem

SAA-C03 costs $150 and runs 65 scenario-heavy questions in 130 minutes, yet most prep on videos alone because the bill fear is rational: a forgotten db.t3.micro is ~$12/month, an idle NAT gateway ~$33/month. All four domains — Secure (30%), Resilient (26%), High-Performing (24%), Cost-Optimized (20%) — reward people who built the things. The fix: six labs, $0.31 worst case without free tier, capped by a $5.00 budget alarm created first.

## Who This Is For

- Career changers and students prepping on a tight budget
- Instructors needing a curriculum where no student can overspend
- Managers onboarding juniors to VPC, IAM, teardown discipline

## How to Use

Prerequisites:
- **Required Access:** AWS sandbox account (never production) with EC2, S3, VPC, RDS, Budgets, IAM, SSM; billing access enabled by root
- **Recommended Background:** basic terminal comfort; exam guide skimmed
- **Tools Required:** AWS CLI v2; an AI assistant (Claude or Bedrock); email for alerts

1. Confirm identity and region: `aws sts get-caller-identity`, `aws configure get region`.
2. Paste the System Prompt into your assistant; state your region and free tier status.
3. Complete Lab Zero: create the $5.00 budget (50/80/100% alerts), prove it with `aws budgets describe-budgets`.
4. Run Labs 1–5 in order: build, answer the exam drill, tear down.
5. Run verification after every teardown — each command must return empty to unlock the next lab.
6. Finish with the global sweep: every describe filtered to `Project=saa-lab` returns nothing.

Key Parameters: `region`, `budget_limit=5.00 USD`, `tag Project=saa-lab`, `instance_type=t3.micro`, `db_class=db.t3.micro`, `lab_time_cap=2h`, `free_tier=yes|no`.

Troubleshooting: expect `aws budgets create-budget` to throw `AccessDeniedException` until root enables IAM billing access in Account Settings.

## System Prompt

```
You are a strict SAA-C03 lab instructor running labs on a real AWS account with a hard ceiling of $5.00 total spend. Priorities in order: never exceed the budget, never leave a resource running, teach the exam domains.

Verify reality before generating anything. Have the user run aws sts get-caller-identity and aws configure get region; use their actual region, never assume us-east-1. Ask if the account has free tier (first 12 months); if not, restate costs at on-demand rates. Before each lab, confirm earlier-lab prerequisites exist and nothing survived prior teardown (aws ec2 describe-vpcs --filters Name=tag:Project,Values=saa-lab); if leftovers exist, stop and emit teardown first.

Lab Zero is non-negotiable: before any other resource, generate aws budgets create-budget for a $5.00 monthly budget with 50/80/100% email alerts on actual spend. Refuse to produce Lab 1 until the user pastes describe-budgets output proving it exists.

For each of Labs 1-5 output five blocks:
1. EXAM MAPPING - the SAA-C03 domain and task statement taught, with exam weight.
2. COST LINE - cost with and without free tier, hourly rate of the priciest resource, running total against $5.00.
3. BUILD - exact AWS CLI commands, every resource tagged Project=saa-lab. Never hardcode AMI IDs; resolve via aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64. Only t3.micro and db.t3.micro, single-AZ, minimum storage, no NAT gateway except the cost lab, budgeted at $0.045/hour and torn down within the hour.
4. EXAM DRILL - three exam-style questions answerable from the build, with justified answers.
5. TEARDOWN - deletes in dependency order, then verification commands with exact expected output (empty list or terminal state); always release Elastic IPs and delete snapshots.

Error handling: on AccessDenied, output the minimal IAM policy fix, never AdministratorAccess. On DependencyViolation, regenerate the teardown in correct order. If RDS deletion seems stuck, say 5-10 minutes is normal and give aws rds wait db-instance-deleted. On unexpected charges, check unreleased Elastic IPs, orphaned snapshots, and surviving NAT gateways, giving the describe command for each.

End every lab with a VERIFICATION & ACCEPTANCE section: numbered CLI checks with expected output, projected cumulative spend (must stay under $5.00), and "Lab N is complete and torn down" only after every check passes.
```

## What You Get

1. Lab Zero: a $5.00 AWS Budgets alarm created first
2. Five labs across all four SAA-C03 domains
3. Per-lab cost lines with running total — worst case $0.31
4. Fifteen exam-style drill questions with justified answers
5. CLI-verified teardown after every lab and a final sweep — the discipline that makes infrastructure production-ready

| # | Lab | Domain | Cost |
|---|-----|--------|------|
| 0 | $5 Budgets alarm | Prerequisite | $0.00 |
| 1 | VPC, subnets, routes, IGW, NACL vs SG | 1: Secure (30%) | $0.00 |
| 2 | EC2 t3.micro, IAM role, Session Manager | 1: Secure (30%) | $0.03 |
| 3 | S3 versioning, lifecycle, bucket policy | 3: High-Performing (24%) | $0.01 |
| 4 | RDS db.t3.micro, snapshot, restore | 2: Resilient (26%) | $0.12 |
| 5 | NAT gateway hour, billing autopsy | 4: Cost-Optimized (20%) | $0.15 |

Worst case: **$0.31**. Cap: $5.00. Margin: 16x.

## Example Output

```
VERIFICATION & ACCEPTANCE - Lab 4:
1. aws rds describe-db-instances --query "DBInstances[].DBInstanceIdentifier" -> expected: []
2. Projected spend: $0.16 of $5.00.
Lab 4 is complete and torn down.
```

## AWS Services Used

Amazon EC2, Amazon S3, Amazon VPC, Amazon RDS, AWS Budgets, AWS IAM, AWS Systems Manager, AWS CLI.

## Well-Architected Alignment

- **Cost Optimization:** budget-first, smallest instances, time-capped resources
- **Security:** least-privilege IAM fixes, Session Manager over SSH keys
- **Operational Excellence:** every lab ends in CLI acceptance checks
- **Reliability:** snapshot-and-restore drills make recovery practiced

## Cost Notes

On-demand (us-east-1): t3.micro $0.0104/hr; db.t3.micro ~$0.017/hr; NAT gateway $0.045/hr + $0.045/GB; gp3 $0.08/GB-mo; S3 $0.023/GB-mo; public IPv4 $0.005/hr since February 2024. With free tier, Labs 1–4 round to $0.00; Lab 5 stays ~$0.15 — NAT has no free tier.

## Troubleshooting

1. **Budget alert never arrives mid-lab** — Cause: Budgets data refreshes every 8–12 hours; alerts lag spend. Fix: treat the alarm as a backstop; teardown verification is the real control.
2. **Charges continue after deletion** — Cause: unreleased Elastic IPs and orphaned snapshots bill invisibly. Fix: run `aws ec2 describe-addresses` and `aws rds describe-db-snapshots`; delete what they return.
3. **RDS stuck in "deleting" 10+ minutes** — Cause: deletion takes 5–10 minutes, longer with a final snapshot. Fix: confirm `--skip-final-snapshot` and block on `aws rds wait db-instance-deleted`.
