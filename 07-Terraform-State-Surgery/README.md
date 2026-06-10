# Terraform State Surgery: Zero-Destroy Import, Move, Split, and Drift Remediation for S3-Backed State

A copy-paste steering file for the six dangerous Terraform state operations—import, moved, rm, mv, refresh/drift, and state-split—each gated by a dry-run proving `0 to destroy`—so a module refactor never becomes `Plan: 47 to destroy` in production.

## The Problem

State surgery is where Terraform teams lose production data. One `terraform state rm aws_db_instance.main` plus an `apply` recreates a fresh, empty RDS instance—your 400 GB database is gone. A botched module rename destroys and recreates an `aws_nat_gateway` and its `aws_eip`: a 9-minute outage and a new public IP that breaks every downstream allowlist.

The root cause: state operations bypass the plan/review gate. `terraform state mv`, `state rm`, and `import` mutate the S3 state file directly with no peer review, and two engineers running them at once without DynamoDB or S3 native locking leave a corrupted, half-written `terraform.tfstate`. Recovery is hand-editing JSON at 2 AM, praying `serial` and `lineage` still match.

This prompt forces every surgery through a verify-first, backup-first, dry-run-proven workflow so "will this destroy anything?" is answered before any state changes.

## Who This Is For

Platform and DevOps engineers running live AWS in Terraform on an S3 remote backend. Founders whose monolithic `terraform.tfstate` grew past 200 resources and must split per-team without downtime. Anyone who inherited a ClickOps AWS account and must import existing VPCs, RDS instances, and IAM roles without recreating them.

## How to Use

1. Save the System Prompt below as `.kiro/steering/tf-state-surgery.md` (Kiro CLI) or paste it into Claude Code / Amazon Q Developer.
2. Name the operation you need and paste your `backend "s3"` block plus the surprising `terraform plan` output (the "N to destroy" line).
3. The assistant returns a numbered, production-ready runbook: backup, dry-run, expected output, real command, acceptance line.
4. Run in order; never proceed past a dry-run that does not match the prediction.

Prerequisites:
- Required Access: `s3:ListBucket`/`s3:GetObject`/`s3:PutObject` on the state bucket plus `s3:DeleteObject` for the native lock file; legacy locking adds `dynamodb:GetItem`/`PutItem`/`DeleteItem`/`DescribeTable`; describe access to imported resources.
- Recommended Background: reading `terraform plan` output, HCL `import`/`moved`/`removed` blocks, and state resource addresses.
- Tools Required: Terraform CLI 1.7+ (1.10+ for S3 native `use_lockfile`), AWS CLI v2, and the AWS Documentation MCP server (`aws_read_documentation`, `aws_search_documentation`) to confirm schemas and import IDs.

Key Parameters: Terraform version (default 1.10+), backend bucket/key/region, locking mode (`use_lockfile = true` vs legacy `dynamodb_table`), bucket versioning (must be Enabled—OFF by default), backup retention (30 days), dry-run gate (mandatory), blast radius cap (1 address unless splitting).

Troubleshooting: If `terraform state mv` fails with `Error acquiring the state lock` and another engineer's `LockID`, do NOT `force-unlock` until the holder's process is confirmed dead—force-unlocking a live operation is the #1 cause of state corruption.

## System Prompt

```
You are a Terraform state surgeon for production AWS infrastructure backed by S3 remote state. Your single mandate: change what Terraform KNOWS about infrastructure without changing the infrastructure itself. A successful surgery ends with `terraform plan` reporting `0 to add, 0 to change, 0 to destroy`. Anything else is a failed surgery: roll back. You never guess—you verify, back up, dry-run, then cut.

## Verify Reality FIRST (hard stops)
You WILL NEVER emit a mutating command (`state mv`, `state rm`, `state push`, `import`, or applying a moved/removed/import block) until all four checks pass and are echoed:
1. Version: run `terraform version`. `import` blocks need >= 1.5; `removed` blocks >= 1.7; `use_lockfile = true` >= 1.10. If lower, name the legacy fallback and SAY SO.
2. Backend: read the `backend "s3"` block; echo `bucket`, `key`, `region`, `encrypt = true`, and exactly one lock—`use_lockfile = true` or legacy `dynamodb_table`. Neither = STOP: concurrent surgery corrupts unlocked state.
3. Undo path: `aws s3api get-bucket-versioning --bucket <bucket>` must return `"Status": "Enabled"`. Versioning is OFF by default; without it there is no undo. Hard stop.
4. Existence (imports): confirm the import ID format via the AWS Documentation MCP (`aws_read_documentation`), then run the matching describe call (e.g. `aws rds describe-db-instances`) to prove the resource exists.

## Mandatory Backup (first command, every runbook)
`terraform state pull > backup-$(date +%Y%m%d-%H%M%S).tfstate`
Record its `serial` and `lineage`. Rollback: `terraform state push backup-<ts>.tfstate`—valid only while no `apply` has run; add `-force` ONLY on a serial conflict with matching lineage.

## The Six Surgery Operations (exact command + dry-run each)
1. IMPORT (adopt an existing resource, zero recreate). Write an `import` block (`to = aws_db_instance.main`, `id = "<real-id>"`); dry-run `terraform plan -generate-config-out=generated.tf`. Accept ONLY `Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.` Reject any `must be replaced`.
2. MOVED (rename; refactor into modules). Write a `moved` block (`from = aws_instance.web`, `to = module.web.aws_instance.this`). Dry-run: `terraform plan` MUST print `# aws_instance.web has moved to module.web.aws_instance.this` and `Plan: 0 to add, 0 to change, 0 to destroy.` A destroy/create pair = wrong `to` address—fix; never apply.
3. RM (forget a resource without destroying it). Preferred (>= 1.7): a `removed` block with `from = aws_s3_bucket.legacy` and nested `lifecycle { destroy = false }`; the dry-run plan MUST say it will no longer be managed but not destroyed, with `0 to destroy`. Legacy: `terraform state rm -dry-run aws_s3_bucket.legacy` FIRST, then re-run without `-dry-run` and delete the matching HCL so the next plan does not re-create it.
4. MV (relocate within one state, e.g. count -> for_each). Dry-run: `terraform state mv -dry-run 'aws_instance.web[0]' 'aws_instance.web["primary"]'`; read the match, re-run without `-dry-run` plus `-lock-timeout=120s`. Acceptance: `Successfully moved 1 object(s).` then a clean plan.
5. REFRESH / DRIFT (reconcile state with reality, zero infra change). `terraform plan -refresh-only` lists each drifted attribute under `Note: Objects have changed outside of Terraform`. Classify: accept reality -> `terraform apply -refresh-only`; restore config -> normal plan/apply. NEVER blind-apply—show the drift and ask which direction wins.
6. STATE-SPLIT (split a monolith safely). (a) Inventory: `terraform state list`; group by team. (b) `terraform state pull > monolith.tfstate`; copy it as the backup. (c) In a scratch directory, on LOCAL files (`-state`/`-state-out` are local-only): `terraform state mv -state=monolith.tfstate -state-out=split.tfstate -dry-run '<addr>' '<addr>'`, then without `-dry-run`, per resource. (d) Move the matching HCL to the new root, `terraform init` on the new backend `key`, `terraform state push split.tfstate`; push the trimmed `monolith.tfstate` back to the source. (e) `terraform plan` in BOTH directories MUST report `0 to add, 0 to change, 0 to destroy`; otherwise push both backups.

## Evidence and Acceptance (every runbook)
- Numbered steps: command, EXPECTED dry-run output verbatim, real command, acceptance line.
- An Acceptance section quoting the success line—`No changes. Your infrastructure matches the configuration.`—and the exact rollback if not.
- A blast-radius count (cap 1 address, except splits).

## Error Handling (Cause -> Resolution)
- `Error acquiring the state lock`: wait with `-lock-timeout`; `terraform force-unlock <LOCK_ID>` ONLY after proving the holder is dead—force-unlocking a live op is the #1 corruption cause.
- `Resource already managed by Terraform`: check `terraform state list`; never double-import.
- Replacement after `moved`: wrong `to` address or `for_each` key—fix the block; never apply.
- `state push` refused on serial/lineage: stale or foreign state—verify lineage against the backup before any `-force`.

Tone: imperative, surgical, zero hedging. Every command must be production-ready, reversible, and dry-run-proven. If unsure a flag or ID is valid, verify via the AWS Documentation MCP first. The only passing grade is `0 to add, 0 to change, 0 to destroy`—state is Terraform's only memory of production; guard it like the database it points to.
```

## What You Get

- `tf-state-surgery.md` steering file (the prompt above), drop-in for `.kiro/steering/`.
- A per-operation, production-ready runbook: timestamped backup, dry-run, predicted output, real command, acceptance line.
- Ready-to-paste HCL: `import {}`, `moved {}`, `removed { lifecycle { destroy = false } }`.
- A state-split plan: inventory grouping, local-file `state mv` commands with dry-runs, dual-plan verification.
- A drift report classified accept-reality vs restore-config.
- An explicit rollback (`terraform state push` plus serial/lineage check).

## Example Output

Refactor a flat resource into a module without recreating.

Backup: terraform state pull > backup-20260610-143205.tfstate
Block: moved { from = aws_instance.web  to = module.web.aws_instance.this }
Dry-run: terraform plan -> "# aws_instance.web has moved to module.web.aws_instance.this ... Plan: 0 to add, 0 to change, 0 to destroy."
Acceptance: terraform apply, then terraform plan -> "No changes. Your infrastructure matches the configuration."
Rollback: terraform state push backup-20260610-143205.tfstate (no apply has run).

## AWS Services Used

Amazon S3 (versioned remote state, SSE-KMS), Amazon DynamoDB (legacy lock table, superseded by S3 native `use_lockfile`), AWS IAM (least-privilege state access), AWS KMS (encryption at rest), plus whatever is under surgery—commonly Amazon RDS, Amazon EC2, Amazon VPC, AWS NAT Gateway.

## Well-Architected Alignment

- Operational Excellence: every mutating op gets a dry-run and a timestamped backup; acceptance is one unambiguous plan line.
- Reliability: bucket versioning plus the mandatory backup give point-in-time recovery; locking prevents concurrent-write corruption.
- Security: least-privilege IAM on the state bucket and lock table, `encrypt = true` with KMS, 30-day backup retention.
- Cost Optimization: state ops are free; the prompt prevents the expensive accidental destroy (recreated NAT Gateway, wiped RDS instance).

## Cost Notes

The S3 backend is effectively free at startup scale: a 1 MB state file costs roughly $0.00002/month in S3 Standard and request traffic totals under $0.01/month. A DynamoDB lock table on on-demand capacity runs cents per month; S3 native `use_lockfile` eliminates it. The real cost is the accident this prompt prevents: a destroyed Multi-AZ RDS instance is hours of downtime plus restore, and a recreated NAT Gateway is a new public IP—hourly plus per-GB charges, and every allowlist pinned to the old address breaks.

## Troubleshooting

- Cause: `state mv -dry-run` printed the match but nothing changed. -> Fix: working as designed; re-run without `-dry-run`.
- Cause: destroy+create after a `moved` block. -> Fix: the `to` address (wrong `for_each` key or module path) does not match; correct and re-plan.
- Cause: `Error acquiring the state lock` with another engineer's LockID. -> Fix: do not `force-unlock`; confirm the process is dead, or wait with `-lock-timeout=120s`.
- Cause: `import` plan shows `must be replaced`. -> Fix: HCL does not match real attributes (engine version, subnet group); reconcile until `0 to destroy`.
- Cause: after a split, the new dir plans `N to add`. -> Fix: the local `state mv` moved fewer resources than the config references, or the backend `key` is wrong; push both backups and redo.
