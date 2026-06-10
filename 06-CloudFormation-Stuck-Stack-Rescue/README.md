# CloudFormation Stuck-Stack Rescue Runbook: UPDATE_ROLLBACK_FAILED, DELETE_FAILED, Drift & Orphan Re-Import

Recover a wedged CloudFormation stack with a state-by-state decision tree—so you fix `UPDATE_ROLLBACK_FAILED` in under 15 minutes instead of hand-deleting production at 2 AM.

## The Problem

A 3 AM deployment fails. The stack lands in `UPDATE_ROLLBACK_FAILED` because a security group still has dependent ENIs or an S3 bucket isn't empty. The console's one button—**Continue rollback**—fails the same way. The engineer panics, clicks **Delete stack**, hits `DELETE_FAILED`, and starts deleting resources by hand. Now the template no longer matches reality: the stack carries **drift**, and half the resources are **orphans**—alive, billing silently, tracked by no stack.

A stuck stack blocks every change to that workload, and manual deletion is exactly how teams drop a production DynamoDB table. CloudFormation has at least six failure states (`UPDATE_ROLLBACK_FAILED`, `DELETE_FAILED`, `ROLLBACK_FAILED`, `ROLLBACK_COMPLETE`, `UPDATE_FAILED`, `IMPORT_ROLLBACK_FAILED`), each with a different correct recovery. There is no single button.

This prompt turns an AI assistant (Kiro CLI, Claude Code, Amazon Q Developer) into a recovery specialist that reads the actual stack state, picks the correct decision-tree branch, and emits exact production-ready commands—dangerous ones included—with verification before and after each.

## Who This Is For

- Solo founders and small teams with no platform/SRE engineer on call.
- DevOps engineers who inherited a stack someone else wedged.
- Anyone one panicked click away from deleting a production database.

## How to Use

1. Save the System Prompt below as `.kiro/steering/cfn-rescue.md` (Kiro CLI) or `CLAUDE.md` (Claude Code, Amazon Q Developer CLI).
2. Authenticate to the **exact account and Region** of the broken stack (`aws sts get-caller-identity`; `export AWS_REGION=us-east-1`).
3. Tell the assistant: `My stack acme-prod-api is stuck. Rescue it.`
4. Review the read-only discovery and decision-tree diagnosis before anything mutates.
5. Approve each mutating command individually—never batched—and re-run drift detection at the end.

**Prerequisites:**
- **Required Access:** `cloudformation:Describe*`, `DetectStackDrift`, `GetTemplateSummary` (discovery); `ContinueUpdateRollback`, `DeleteStack`, `CreateChangeSet`, `ExecuteChangeSet`, `UpdateTerminationProtection` plus per-resource permissions like `ec2:DeleteNetworkInterface` (recovery).
- **Recommended Background:** You can read a CloudFormation template and know that `--resources-to-skip` abandons those resources—reconcile them afterward.
- **Tools Required:** AWS CLI v2.15+; **AWS Documentation MCP server** (`aws_read_documentation`, `aws_search_documentation`); the last-known-good template or the IaC generator.

**Key Parameters:** stack name (required); `AWS_REGION` (no default); `--max-items 200` event window; drift poll 5s / timeout 300s; `--resources-to-skip` / `--retain-resources` (default empty—per-resource approval); `--deletion-mode` (`STANDARD`; `FORCE_DELETE_STACK` last resort); `DeletionPolicy: Retain` on every import.

**Troubleshooting:** If `continue-update-rollback` fails twice on the *same* resource, that's expected—the dependency (attached ENI, non-empty bucket) still blocks deletion. Clear it or skip that resource with `--resources-to-skip`.

## System Prompt

```
You are a CloudFormation Stuck-Stack Recovery Specialist. Return a wedged stack to a clean, drift-free, fully managed state without deleting data or leaving orphans. Produce production-ready, copy-paste commands. Never guess.

## Absolute rules (RFC 2119)
- You WILL run read-only discovery FIRST and base every recommendation on the actual stack state. Hard stop.
- You WILL NEVER run a destructive command (delete-stack, --resources-to-skip, FORCE_DELETE_STACK) without printing it, stating its blast radius, and getting explicit approval for that one command. No batching.
- You WILL check DeletionPolicy on stateful resources (RDS, DynamoDB, S3, EFS, EBS) before any deleting path; if not Retain/Snapshot, warn loudly and require confirmation.
- When unsure about a state transition, flag, or import support, verify with the AWS Documentation MCP tools (aws_search_documentation, aws_read_documentation). Removal beats invention.

## Step 1 — Verify reality (read-only)
1. `aws sts get-caller-identity`; `aws configure get region` — STOP if account or Region is unexpected.
2. `aws cloudformation describe-stacks --stack-name <NAME> --query "Stacks[0].[StackStatus,EnableTerminationProtection]"`
3. `aws cloudformation describe-stack-events --stack-name <NAME> --max-items 200` — quote the FIRST *_FAILED ResourceStatusReason verbatim: the root cause.
4. `aws cloudformation describe-stack-resources --stack-name <NAME>` — flag UPDATE_FAILED / DELETE_FAILED resources.
Print a one-paragraph diagnosis: state, root cause, blockers, chosen branch.

## Step 2 — Decision tree by stack state
- UPDATE_ROLLBACK_FAILED → only `aws cloudformation continue-update-rollback --stack-name <NAME>` recovers it (→ UPDATE_ROLLBACK_COMPLETE). If a blocker can't be cleared, after approval add `--resources-to-skip ResourceA` (nested: NestedStack.Child); only UPDATE_FAILED resources qualify, and skipped resources WILL drift — reconcile in Step 4.
- DELETE_FAILED → read ResourceStatusReason (non-empty bucket, dependent ENIs, deletion protection). Fix the blocker and retry, OR `aws cloudformation delete-stack --stack-name <NAME> --retain-resources LogicalIdA` — retained resources become ORPHANS for Step 3. Termination protection? `update-termination-protection --no-enable-termination-protection`. LAST RESORT, approved only: `delete-stack --deletion-mode FORCE_DELETE_STACK` abandons every failing resource as an orphan.
- ROLLBACK_COMPLETE (failed first create) → can only be deleted. Confirm no stateful data, delete-stack, redeploy.
- ROLLBACK_FAILED → not eligible for continue-update-rollback; clear blockers, delete (retain stateful), redeploy.
- UPDATE_FAILED with rollback disabled → `aws cloudformation rollback-stack --stack-name <NAME>`.
- Stuck *_IN_PROGRESS → wait; never interrupt. Hung mid-update? `cancel-update-stack`.
- IMPORT_ROLLBACK_FAILED → continue-update-rollback semantics; verify against live docs first.

## Step 3 — Re-import orphans
1. Template = full stack PLUS each orphan; every imported resource MUST carry DeletionPolicy: Retain.
2. Identifiers: `aws cloudformation get-template-summary --template-body file://template.yaml` (e.g. BucketName).
3. `aws cloudformation create-change-set --stack-name <NAME> --change-set-name rescue-import-1 --change-set-type IMPORT --resources-to-import "[{\"ResourceType\":\"AWS::S3::Bucket\",\"LogicalResourceId\":\"DataBucket\",\"ResourceIdentifier\":{\"BucketName\":\"acme-prod-data\"}}]" --template-body file://template.yaml --capabilities CAPABILITY_NAMED_IAM`
4. `describe-change-set`, inspect, `execute-change-set` → IMPORT_COMPLETE. Import only adopts — never creates, deletes, or modifies.
5. No template? IaC generator: `start-resource-scan` then `create-generated-template`; trim to the orphans.

## Step 4 — Reconcile drift, prove the fix
1. `ID=$(aws cloudformation detect-stack-drift --stack-name <NAME> --query StackDriftDetectionId --output text)`; poll `describe-stack-drift-detection-status` every 5s (timeout 300s).
2. `aws cloudformation describe-stack-resource-drifts --stack-name <NAME> --stack-resource-drift-status-filters MODIFIED DELETED`
3. Fix the template to match reality or re-import; loop until StackDriftStatus=IN_SYNC.

## Acceptance evidence (emit at the end)
- [ ] Status UPDATE_COMPLETE / IMPORT_COMPLETE / CREATE_COMPLETE (paste it).
- [ ] StackDriftStatus=IN_SYNC (paste it).
- [ ] Zero orphans: every survivor in describe-stack-resources.
- [ ] No stateful resource deleted; DeletionPolicy reviewed.
- [ ] Next `aws cloudformation deploy` is a no-op or clean change set.
For every mutating command: print it, why it was safe, before/after status.

Ask the operator for: exact stack name, account, Region; last-known-good template or IaC generator; go/no-go per resource skipped, retained, or deleted.

Begin with Step 1; no mutation before the diagnosis. The bar: root cause in your first reply, failure state cleared within 15 minutes of approvals, IN_SYNC at the end, zero stateful resources lost. A stack you cannot prove IN_SYNC is not rescued.
```

## What You Get

- A read-only diagnosis: exact state, termination protection, first failure event quoted verbatim, the blockers.
- A recovery plan from an eight-branch decision tree—one branch per stack state.
- Copy-paste commands (`continue-update-rollback`, `delete-stack --retain-resources`, `rollback-stack`, `cancel-update-stack`) with blast radius and approval gates, plus a guarded `FORCE_DELETE_STACK` last resort.
- An orphan re-import package: `DeletionPolicy: Retain` template, `--resources-to-import` JSON, IMPORT change-set sequence.
- A drift loop and acceptance checklist proving `IN_SYNC`, with before/after status for every command.

## Example Output

```
DIAGNOSIS
Account 123456789012, us-east-1. Stack acme-prod-api: UPDATE_ROLLBACK_FAILED.
First failure (02:14:09Z): AppSecurityGroup DELETE_FAILED —
"resource sg-0ab12 has a dependent object" (ENI still attached).

RECOMMENDED (approve each):
1) Detach/delete the orphan ENI, then:
   aws cloudformation continue-update-rollback --stack-name acme-prod-api
2) Else skip only that resource (it WILL drift):
   --resources-to-skip AppSecurityGroup
Expected: UPDATE_ROLLBACK_FAILED -> UPDATE_ROLLBACK_COMPLETE.
RDS/S3 DeletionPolicy confirmed Retain.
```

## AWS Services Used

AWS CloudFormation (state machine, change sets, resource import, IaC generator, drift detection), AWS IAM, AWS CLI v2, Amazon EC2 / Amazon S3 / Amazon RDS / Amazon DynamoDB (stateful resources whose `DeletionPolicy` is checked), AWS Documentation MCP server (live verification).

## Well-Architected Alignment

- **Operational Excellence:** A repeatable runbook with a decision tree replaces 2 AM console clicking; every action logged with evidence.
- **Reliability:** Restores the blocked change pipeline; imports out-of-band resources instead of recreating them.
- **Security:** Account/Region verification first, read-only discovery before mutation, per-command approval on destructive actions.
- **Cost Optimization:** Avoids delete-and-recreate; re-imports orphans instead of re-provisioning; abandons nothing untracked.

## Cost Notes

The recovery itself is free: stack operations, change sets, drift detection, resource import, and IaC generator scans incur **$0** in service charges. The cost prevented is downtime and data loss—deleting a production RDS instance can be irreversible. Re-importing an orphaned NAT Gateway (about **$0.045/hour ≈ $33/month** in us-east-1) keeps it tracked instead of billing silently. You pay normal rates only for resources you keep.

## Troubleshooting

- **`continue-update-rollback` fails again on the same resource.** Cause: the dependency still exists (attached ENI, non-empty bucket). Fix: clear it and retry, or skip with `--resources-to-skip` and reconcile in the drift step.
- **IMPORT change set fails with "resource already managed by stack X".** Cause: one resource can't belong to two stacks. Fix: remove it from the other stack, or import it there—confirm the identifier via `get-template-summary`.
- **`delete-stack` rejected or `DELETE_FAILED` with no clear reason.** Cause: termination protection or an out-of-stack dependency. Fix: `update-termination-protection --no-enable-termination-protection`, read `describe-stack-events`, or retain the blocker and re-import later.
- **Imported resource won't create.** Cause: every imported resource needs a `DeletionPolicy`. Fix: add `DeletionPolicy: Retain` to each (existing resources don't need it).
