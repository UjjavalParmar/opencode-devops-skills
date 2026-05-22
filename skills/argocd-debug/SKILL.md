---
name: argocd-debug
description: Debug ArgoCD applications — OutOfSync, Degraded, sync hooks, ApplicationSet generation, repo-server/redis/application-controller errors, RBAC, Project restrictions, SSO. Use for anything under argoproj.io CRDs.
---

# Order of operations (read-only)
1. `argocd app get <APP> -o wide`
2. `argocd app diff <APP>` — drift vs Git
3. `argocd app history <APP>` ; `argocd app manifests <APP> --revision <REV>`
4. `kubectl -n argocd get applications.argoproj.io <APP> -o yaml | yq '.status'`
5. Controller logs: `kubectl -n argocd logs deploy/argocd-application-controller --tail=200`
6. Repo-server: `kubectl -n argocd logs deploy/argocd-repo-server --tail=200`

# Symptom → first check
- **OutOfSync** → `argocd app diff`; ignoreDifferences / spec.syncPolicy.
- **ComparisonError / Unknown** → repo-server logs; private repo creds: `argocd repo list`.
- **SyncFailed (hook)** → `argocd app get <APP>` resources; `kubectl logs` of PreSync/PostSync job.
- **RBAC: permission denied** → `argocd account can-i <action> <resource> <project>/<APP>`; check `argocd-rbac-cm`.
- **ApplicationSet not generating** → `kubectl -n argocd logs deploy/argocd-applicationset-controller`.
- **Stuck Progressing** → resource health checks; custom Lua in `argocd-cm` `resource.customizations`.
- **AutoSync not firing** → `spec.syncPolicy.automated`; `.spec.syncPolicy.automated.selfHeal`.

# Mutations (require confirmation)
- Manual sync: `argocd app sync <APP>` — rollback: `argocd app rollback <APP> <REV>`
- Hard refresh: `argocd app get <APP> --hard-refresh` (read-only effectively)
- Terminate op: `argocd app terminate-op <APP>`
- **Never** `argocd app delete <APP> --cascade` without confirming downstream resources.

# Verify
`argocd app wait <APP> --health --timeout 300`
