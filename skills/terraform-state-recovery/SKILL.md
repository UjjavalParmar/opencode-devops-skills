---
name: terraform-state-recovery
description: Recover Terraform state — stuck locks, corrupted/missing state, resources drifted out of state, accidental destroy, backend migration. HIGH-RISK skill — every operation must back up state first. Use ONLY for state-level issues; for normal change review use terraform-review.
---

# Preconditions
- Confirm workspace + backend: `terraform workspace show` ; `terraform version` ; check `backend` block in code.
- Confirm AWS account/region: `aws sts get-caller-identity` ; `echo $AWS_REGION`.
- Verify nobody else is mid-apply (CI runs, other engineers). If unsure, STOP.

# Escalation
- Stateful resource (RDS, EBS, S3 with data) at risk → page DBA / data owner before any `state rm`, `import`, or `force-unlock`.
- Lock held by an active CI run → contact pipeline owner; do NOT force-unlock.
- State file lost AND no versioning → escalate to senior; recovery may require rebuild-from-config + re-import per resource.

# RULE 0 — back up state first, every time
```
terraform state pull > state.backup.$(date -u +%Y%m%dT%H%M%SZ).json
```
For S3 backend: also `aws s3 cp s3://<B>/<K> ./state.s3.backup.json` and enable bucket versioning.

# Locked state (DynamoDB / backend lock)
1. Identify holder — error shows `ID`, `Who`, `Created`, `Operation`.
2. Confirm no other apply is running (CI, other engineer).
3. `terraform force-unlock <LOCK-ID>` — destructive if a real apply is in flight. Confirm via CI/CloudTrail first.

# Corrupted / missing state
- Restore from versioned backend: `aws s3api list-object-versions --bucket <B> --prefix <K>` → `aws s3api get-object --version-id <V> --bucket <B> --key <K> state.json`
- Push restored state: `terraform state push state.json` (matching serial or higher)

# Resource exists in cloud, not in state (drift / lost)
```
terraform import <ADDR> <CLOUD-ID>
terraform plan        # must show 0 changes (or only metadata)
```
If plan shows replace ⇒ stop, fix attributes first.

# Resource in state, gone from cloud
```
terraform state rm <ADDR>   # then next apply recreates if still in config
```

# Move / rename
```
terraform state mv <SRC-ADDR> <DST-ADDR>
```

# Backend migration
```
terraform init -migrate-state -backend-config=...
```
Keep old backend read-only until verified.

# Accidental destroy in last apply
- If backend versioned: restore prior state (above), `terraform plan` will recreate resources from config. For stateful resources (RDS, EBS) restore from AWS snapshot/backup BEFORE plan.

# Verify
`terraform plan` — expect "No changes" or only the intended diff.

# Never
- `terraform apply -refresh-only` on a broken state without backup.
- Hand-edit state JSON unless backend versioning is confirmed and a backup exists.
