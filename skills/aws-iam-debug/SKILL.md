---
name: aws-iam-debug
description: Debug AWS IAM authorization failures — AccessDenied, AssumeRole errors, SCP/permission-boundary denials, IRSA / Pod Identity for EKS, cross-account access, KMS key policies. Use for any 403 or `not authorized to perform` error from AWS APIs.
---

# Preconditions
- Confirm account + role: `aws sts get-caller-identity`. Wrong account = wrong policies.
- CloudTrail enabled in the region of the failing call (otherwise `lookup-events` returns nothing).

# Escalation
- SCP denial in Organizations → only the Org management account can amend; escalate, do not work around with cross-account assume.
- KMS key policy denial on a CMK owned by another team → key owner must edit; never grant via IAM-only.
- Production role trust policy edits → require change approval; stage in a non-prod role first.

# Identify caller + action
```
aws sts get-caller-identity
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<API> --max-results 5
```
Read the AccessDenied message verbatim — it names principal, action, resource, and (often) the denying policy type.

# Simulate (read-only, authoritative)
```
aws iam simulate-principal-policy \
  --policy-source-arn <PRINCIPAL-ARN> \
  --action-names <svc>:<Action> \
  --resource-arns <RES-ARN>
```
Reports `allowed/explicitDeny/implicitDeny` + which statement.

# Denial layers (check all)
1. Identity policy (role/user)
2. Resource policy (S3 bucket, KMS, SQS, Lambda, ECR)
3. Permission boundary
4. SCP (Organizations) — `aws organizations describe-effective-policy --policy-type SERVICE_CONTROL_POLICY --target-id <ACCT>`
5. Session policy (assume-role `--policy`)
6. VPC endpoint policy

# AssumeRole failures
```
aws sts assume-role --role-arn <R> --role-session-name dbg
```
Check trust policy: `aws iam get-role --role-name <R> --query 'Role.AssumeRolePolicyDocument'`.
- External ID mismatch
- `sts:AssumeRoleWithWebIdentity` (IRSA): condition keys `aud`, `sub`
- Source IP / VPCE conditions

# EKS IRSA
- ServiceAccount annotation: `eks.amazonaws.com/role-arn`
- Role trust must allow `<OIDC-PROVIDER>:sub` = `system:serviceaccount:<NS>:<SA>`
- Pod env: `AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`

# EKS Pod Identity (newer)
```
aws eks list-pod-identity-associations --cluster-name <C>
```

# Access Analyzer (preview reachability)
```
aws accessanalyzer list-analyzers
aws accessanalyzer get-finding --analyzer-arn <A> --id <F>
```

# Mutations
Policy edits: stage in dev, use `--generate-cli-skeleton`, attach with version, keep prior version (`SetDefaultPolicyVersion` rollback).
