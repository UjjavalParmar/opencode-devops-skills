---
name: eks-debug
description: Debug Amazon EKS — node group / Karpenter scaling, VPC CNI IP exhaustion, kubelet/auth issues, aws-auth ConfigMap or Access Entries, control-plane endpoint access, addon failures (CoreDNS, kube-proxy, EBS/EFS CSI). Use for cluster-level EKS problems. For workload bugs use kubernetes-debug; for IAM use aws-iam-debug.
---

# Preconditions
- Confirm cluster: `kubectl config current-context` ; `aws sts get-caller-identity`.
- kubeconfig fresh: `aws eks update-kubeconfig --name <C> --region <R>` if context is stale.
- Versions: `kubectl version` (skew ≤ ±1 minor from control plane).

# Escalation
- Control-plane endpoint unreachable AND change in last 24h to VPC / endpoint access → escalate to network owner before flipping `endpointPrivateAccess`.
- Karpenter mass-disruption (>50% of nodes draining) → pause: `kubectl annotate nodepool <NP> karpenter.sh/disrupted=paused` and escalate before further action.
- Addon downgrade or version-skip across >1 minor → coordinate with platform team; do not `--force` resolve-conflicts on prod.

# Cluster facts
```
aws eks describe-cluster --name <C> --query 'cluster.{ver:version,ep:endpoint,access:resourcesVpcConfig,status:status,plat:platformVersion}'
aws eks list-addons --cluster-name <C>
aws eks describe-addon --cluster-name <C> --addon-name <A>
kubectl version
kubectl get nodes -o wide
```

# Authentication
- Access Entries (current): `aws eks list-access-entries --cluster-name <C>` ; `aws eks describe-access-entry --cluster-name <C> --principal-arn <P>`
- Legacy `aws-auth` CM: `kubectl -n kube-system get cm aws-auth -o yaml`
- kubeconfig: `aws eks update-kubeconfig --name <C> --region <R>`

# Nodes not joining
1. `aws eks describe-nodegroup --cluster-name <C> --nodegroup-name <NG>` — health.issues
2. Node role has `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly`, `AmazonEKS_CNI_Policy` (or IRSA on aws-node)
3. SG: node↔control-plane 443; node↔node all
4. Subnet route to NAT/IGW; cluster endpoint reachable (private vs public)
5. SSM to node: `aws ssm start-session --target <I>` → `journalctl -u kubelet -n 200`

# VPC CNI IP exhaustion
```
kubectl -n kube-system logs ds/aws-node --tail=200
kubectl -n kube-system describe ds aws-node | grep -E 'WARM|MINIMUM|MAX_ENI|PREFIX'
```
Mitigate: prefix delegation (`ENABLE_PREFIX_DELEGATION=true`), larger instance, more subnet CIDR.

# Karpenter
```
kubectl get nodepool,ec2nodeclass
kubectl -n kube-system logs deploy/karpenter --tail=200
kubectl get nodeclaim -o wide
```
Pending pods: `kubectl describe pod <P>` — Karpenter logs the disruption/provisioning decision.

# Addons
```
aws eks describe-addon --cluster-name <C> --addon-name vpc-cni
aws eks update-addon --cluster-name <C> --addon-name vpc-cni --addon-version <V> --resolve-conflicts PRESERVE
```
Rollback: `--addon-version <PREV>`.

# Verify
`kubectl get nodes` ; `kubectl run net --image=busybox --restart=Never -it --rm -- wget -qO- https://kubernetes.default`
