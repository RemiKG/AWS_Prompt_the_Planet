# IAM Least-Privilege Katas

Five graded policy-repair drills in a sandbox account — `simulate-principal-policy` is the only judge, and you pass all 5 in one afternoon for $0.00.

## The Problem

A single `"Action": "s3:*"` statement grants over 100 distinct S3 permissions when the workload needs 2. Teams ship `Resource: "*"` to unblock a sprint, and one leaked key plus `iam:PassRole` on `*` becomes an account-takeover path. Reading does not build the muscle; repairing five broken policies under a hard grading gate does. This is that production-ready practice loop — every API it touches is free.

## Who This Is For

- Developers who freeze when a policy JSON appears in review
- Engineers prepping for security reviews or the Security Specialty exam
- Team leads who want a zero-cost IAM onboarding drill

## How to Use

**Prerequisites:**
- **Required Access:** a sandbox identity with `iam:CreateUser`, `iam:PutUserPolicy`, `iam:GetUser`, `iam:SimulatePrincipalPolicy`, `cloudtrail:LookupEvents`. Never run in production.
- **Recommended Background:** basic JSON.
- **Tools Required:** AWS CLI v2; any AI assistant that accepts a system prompt (Amazon Q Developer, Claude).

1. Confirm the sandbox: `aws sts get-caller-identity`.
2. Paste the System Prompt; it verifies account, region, and practice user, then deals Kata 1.
3. Write your fix, apply with `aws iam put-user-policy`, paste it back.
4. Run the grading command the assistant builds; paste the JSON output.
5. Pass = required actions `allowed`, forbidden ones `implicitDeny`.
6. Review the acceptance table; clean up with `aws iam delete-user-policy` then `delete-user`.

**Key Parameters:** `--policy-source-arn` (user ARN), `--action-names` (graded actions), `--resource-arns` (need not exist), `--context-entries` (condition keys, katas 2 and 4).

**Troubleshooting:** `NoSuchEntity` means your fix was never applied — re-run `aws iam put-user-policy --user-name kata-student --policy-name kataN --policy-document file://policy.json`, re-grade.

## System Prompt

```
You are an IAM least-privilege dojo master grading a five-kata series in a
sandbox AWS account. Each kata, easy to hard, centers on one real
overbroad-policy flaw; grade every fix with aws iam
simulate-principal-policy before declaring a pass.

Verify reality before generating anything. Have the learner run aws sts
get-caller-identity and confirm the account is a sandbox, never production.
Run aws iam get-user --user-name kata-student; if missing, output the exact
aws iam create-user command. Confirm the region with aws configure get
region. Use the real account ID and region in every ARN; never invent
resource names — derive them from the account ID
(kata-bucket-<ACCOUNT_ID>).

Run in strict order — flaw, then pass gate:
1. S3 wildcard — s3:* on Resource "*". Pass: s3:GetObject and s3:ListBucket
   on one bucket and its objects; s3:DeleteObject denies.
2. EC2 blast radius — ec2:* on "*". Pass: StartInstances and StopInstances
   only where ec2:ResourceTag/env = sandbox, plus DescribeInstances;
   TerminateInstances denies.
3. PassRole escalation — iam:PassRole on "*". Pass: PassRole on one role
   ARN with iam:PassedToService = ec2.amazonaws.com; any other role denies.
4. DynamoDB tenant leak — dynamodb:* on all tables. Pass: GetItem and Query
   on one table with dynamodb:LeadingKeys = ${aws:username}; Scan denies.
5. NotAction trap — Allow with NotAction iam:* on "*", which grants nearly
   everything else. Pass: explicit Allows covering only what katas 1-4
   need; sts:AssumeRole and kms:Decrypt both deny.

For each kata: show the broken policy JSON and name the flaw in one
sentence; state pass criteria as exact action/resource pairs; wait for the
fixed policy; grade only via the exact simulate-principal-policy command
with --policy-source-arn, --action-names, --resource-arns, and
--context-entries where conditions apply — required actions must return
"allowed", forbidden ones "implicitDeny" or "explicitDeny". On pass, give a
one-line aws cloudtrail lookup-events follow-up for kata-student.

Error handling: on NoSuchEntity, give the exact aws iam put-user-policy
command to apply the fix. If a forbidden action returns allowed, quote the
offending Sid and name the wildcard or missing condition — never just say
try again. On AccessDenied against the simulator, output the minimal policy
granting iam:SimulatePrincipalPolicy. If output is ambiguous, request raw
JSON before grading.

Never mark a kata passed without pasted simulator output. End every session
with a Verification and Acceptance section: all five katas, grading command,
evalDecision per action, pass/fail, cleanup commands, and total cost, which
must read $0.00 — IAM, the simulator, STS, and CloudTrail Event history are
all free.
```

## What You Get

1. Five katas, easy to hard: S3 wildcard, EC2 blast radius, PassRole escalation, DynamoDB tenant leak, NotAction trap
2. An exact `simulate-principal-policy` grading command per kata
3. Failure feedback quoting the offending `Sid` and the exact flaw
4. A CloudTrail `lookup-events` follow-up per kata
5. A final acceptance table: kata, command, evalDecision per action, pass/fail, $0.00 total

## Example Output

```
KATA 1 — S3 WILDCARD
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::111122223333:user/kata-student \
  --action-names s3:GetObject s3:ListBucket s3:DeleteObject \
  --resource-arns arn:aws:s3:::kata-bucket-111122223333/*
Result: DeleteObject=allowed -> FAIL
Sid "S3Read": "s3:*" still wildcards delete. Fix, re-apply, re-grade.
```

## AWS Services Used

- **AWS IAM** — practice user and inline policies
- **IAM Policy Simulator** (`iam simulate-principal-policy`) — the grading gate
- **AWS CloudTrail** — Event history
- **AWS STS** and **AWS CLI v2** — `get-caller-identity` reality check

## Well-Architected Alignment

- **Security:** SEC03 — grant least privilege; every kata drills exactly this
- **Operational Excellence:** graded drills make policy review a learned operation
- **Cost Optimization:** runs entirely on free APIs; $0.00 by design

## Cost Notes

$0.00. IAM, the simulator, and STS are free; CloudTrail Event history (90 days of management events) is free. The simulator evaluates policy logic against ARNs without touching real resources — the only artifact is one free IAM user.

## Troubleshooting

1. **Simulator says `allowed` for a denied action.** Cause: another policy on the user still grants it. Fix: `aws iam list-user-policies` and `list-attached-user-policies`, remove stragglers, re-grade.
2. **`AccessDenied` calling `SimulatePrincipalPolicy`.** Cause: your grader identity (not the practice user) lacks that action. Fix: attach `iam:SimulatePrincipalPolicy` to your own identity, retry.
3. **Condition katas (2 and 4) always `implicitDeny`.** Cause: the simulator lacks the condition's context keys. Fix: add `--context-entries ContextKeyName=ec2:ResourceTag/env,ContextKeyValues=sandbox,ContextKeyType=string`.
