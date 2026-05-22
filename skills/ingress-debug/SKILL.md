---
name: ingress-debug
description: Debug Kubernetes Ingress / Gateway API traffic — 404/502/503/504 from ingress, TLS/cert-manager issues, path/host routing, ingress-nginx and AWS Load Balancer Controller specifics. Use when traffic flows through an Ingress, IngressClass, or Gateway. NOT for in-cluster Service DNS — use dns-debug.
---

# Layered check (outside → in)
1. **DNS** → `dig +short <HOST>` ; expected LB hostname/IP
2. **LB reachable** → `curl -vk https://<HOST>/ -H 'Host: <HOST>'`
3. **Ingress object** → `kubectl get ingress -A -o wide` ; `kubectl describe ingress <I> -n <NS>` (events, address)
4. **Backend Service** → `kubectl get svc <S> -n <NS>` ; endpoints: `kubectl get endpointslices -n <NS> -l kubernetes.io/service-name=<S>`
5. **Pods Ready** → `kubectl get pod -n <NS> -l <SEL>`

# Code → likely cause
- **404 from nginx** → host/path mismatch, wrong `ingressClassName`, default backend
- **502** → upstream pod crashing, wrong target port, TLS to backend misconfigured
- **503** → no ready endpoints (`kubectl get endpoints <S> -n <NS>`)
- **504** → backend timeout; tune `nginx.ingress.kubernetes.io/proxy-{read,send}-timeout`
- **TLS untrusted** → cert-manager: `kubectl describe certificate <C> -n <NS>` ; `kubectl get certificaterequest,order,challenge -n <NS>`

# ingress-nginx
```
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller --tail=200
kubectl -n ingress-nginx exec deploy/ingress-nginx-controller -- nginx -T | grep -A5 '<HOST>'
```

# AWS Load Balancer Controller (ALB)
```
kubectl -n kube-system logs deploy/aws-load-balancer-controller --tail=200
kubectl describe ingress <I> -n <NS>   # ALB ARN in annotations
aws elbv2 describe-target-health --target-group-arn <TG-ARN>
```
Unhealthy targets ⇒ check `healthcheck-path` annotation and SG ingress to nodePort/pod IP.

# Gateway API
```
kubectl get gateway,httproute,referencegrant -A
kubectl describe httproute <R> -n <NS>   # conditions: Accepted, ResolvedRefs
```

# Verify
`curl -v --resolve <HOST>:443:<LB-IP> https://<HOST>/<PATH>`
