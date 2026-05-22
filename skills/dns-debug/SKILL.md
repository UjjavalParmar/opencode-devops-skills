---
name: dns-debug
description: Debug DNS resolution — in-cluster (CoreDNS, Service/Pod DNS, ndots, search domains, NXDOMAIN, SERVFAIL) and host-level (/etc/resolv.conf, systemd-resolved, Route53). Use when name resolution fails or is slow. NOT for ingress hostname / cert issues — use ingress-debug.
---

# In-cluster
```
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=200
kubectl -n kube-system get cm coredns -o yaml         # Corefile
```

Test from a pod:
```
kubectl run -it --rm dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.7 --restart=Never -- sh
# inside:
cat /etc/resolv.conf
nslookup kubernetes.default
nslookup <svc>.<ns>.svc.cluster.local
dig +trace example.com
```

# Symptom → cause
- **NXDOMAIN for Service** → wrong namespace, Service not created, headless w/o endpoints
- **Slow lookups (~5s)** → conntrack race; set `ndots:1` or use FQDN; consider NodeLocal DNSCache
- **SERVFAIL** → CoreDNS upstream broken; check `forward . /etc/resolv.conf` target
- **Resolves on node but not pod** → CoreDNS down; check NetworkPolicy blocking UDP/53 to kube-dns
- **External resolves intermittently** → CoreDNS cache plugin / upstream rate-limit

# Host-level
```
resolvectl status                          # systemd-resolved
cat /etc/resolv.conf
dig @<NS> example.com +short
getent hosts example.com
```

# AWS Route53 / VPC DNS
```
aws route53 list-resource-record-sets --hosted-zone-id <Z>
aws route53 test-dns-answer --hosted-zone-id <Z> --record-name <N> --record-type A
```
VPC: `enableDnsSupport` + `enableDnsHostnames`; Resolver endpoints for hybrid.

# Verify
`dig +short <NAME> @<RESOLVER>` from the failing host/pod.
