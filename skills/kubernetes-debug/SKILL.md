---
name: kubernetes-debug
description: Debug Kubernetes workload/cluster issues — CrashLoopBackOff, ImagePullBackOff, Pending pods, OOMKilled, evictions, RBAC denials, node NotReady, control-plane errors. NOT for ingress/DNS/Helm/ArgoCD specific issues — use those skills.
---

# Order of operations (read-only)
1. `kubectl get pod <POD> -n <NS> -o wide`
2. `kubectl describe pod <POD> -n <NS>` — events at bottom
3. `kubectl logs <POD> -n <NS> --all-containers --tail=200`
4. `kubectl logs <POD> -n <NS> --previous` (if restarted)
5. `kubectl get events -n <NS> --sort-by=.lastTimestamp | tail -30`

# Symptom → first command
- **Pending** → `kubectl describe pod` (FailedScheduling reason); `kubectl describe node <N>` (taints, allocatable).
- **ImagePullBackOff** → `kubectl describe pod`; verify `imagePullSecrets`.
- **CrashLoopBackOff** → `--previous` logs; `kubectl get pod -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'`.
- **OOMKilled** → `terminated.reason=OOMKilled`; raise `resources.limits.memory`.
- **RBAC denied** → `kubectl auth can-i <verb> <res> -n <NS> --as=<SA>`.
- **Node NotReady** → `kubectl describe node`; `kubectl get --raw=/readyz?verbose`.
- **Stuck Terminating** → `kubectl get <res> <n> -o jsonpath='{.metadata.finalizers}'`.

# Live debug (non-destructive)
- Ephemeral container: `kubectl debug -it <POD> -n <NS> --image=busybox --target=<container>`
- Node shell: `kubectl debug node/<N> -it --image=busybox`
- Port-forward: `kubectl port-forward -n <NS> <POD> <L>:<R>`

# Mutations (require confirmation)
- Restart: `kubectl rollout restart deploy/<D> -n <NS>` — rollback: `kubectl rollout undo deploy/<D> -n <NS>`
- Scale: `kubectl scale deploy/<D> --replicas=<N> -n <NS>` — rollback: previous count
- Delete pod: controller recreates; no rollback needed

# Verify
`kubectl rollout status deploy/<D> -n <NS> --timeout=2m`
