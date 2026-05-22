---
name: terraform-review
description: Review Terraform code or plan output for safety, destructive changes, drift, and provider/version issues. Use BEFORE `terraform apply`. For corrupted/locked/missing state, use terraform-state-recovery.
---

# Pre-flight
```
terraform fmt -recursive -check
terraform validate
terraform providers
terraform plan -out tfplan
terraform show -json tfplan | jq '.resource_changes[] | {addr:.address, actions:.change.actions}'
```

# Destructive markers (flag every one)
- `actions` contains `delete` or `["delete","create"]` (replace)
- `replace_paths` non-empty
- Removals of: `aws_db_instance`, `aws_rds_cluster`, `aws_s3_bucket`, `aws_kms_key`, `aws_iam_role`, `aws_eks_cluster`, `aws_ebs_volume`, anything with `prevent_destroy=false` on stateful data
- `force_new` attribute changes on stateful resources

# Review checklist
- `lifecycle { prevent_destroy = true }` on stateful resources
- `create_before_destroy` where downtime matters
- Provider version pinned (`required_providers` with `~>`)
- Backend with locking (S3+DynamoDB, or `terraform_remote_state` consumers documented)
- No hard-coded secrets — use `aws_secretsmanager_secret_version` / SSM
- `count`/`for_each` keyed by stable IDs (not list index where order changes)
- `sensitive = true` on secret outputs

# Safe apply pattern
```
terraform plan -out tfplan
terraform show tfplan          # human review
terraform apply tfplan         # apply EXACT plan
```
Rollback: `git revert` the change + `terraform apply`. State-level rollback only via state-recovery skill.

# Targeted apply (last resort)
`terraform apply -target=<addr>` — note in plan that subsequent full apply is required.
