---
name: helm-debug
description: Debug Helm releases — failed install/upgrade, pending-upgrade/pending-rollback state, template rendering errors, values precedence, hook failures, chart dependency issues, repo auth. Use for anything driven by `helm` CLI or Helm-managed releases.
---

# Inspect (read-only)
```
helm list -A
helm status <REL> -n <NS>
helm get values <REL> -n <NS> -a        # all, including computed
helm get manifest <REL> -n <NS>
helm history <REL> -n <NS>
```

# Template / values issues
```
helm template <REL> <CHART> -f values.yaml --debug 2>&1 | less
helm lint <CHART>
helm show values <CHART> --version <V>
```

# Diff before upgrade (requires helm-diff plugin)
```
helm diff upgrade <REL> <CHART> -n <NS> -f values.yaml --version <V>
```

# Symptom → action
- **`UPGRADE FAILED: another operation in progress`** → check `helm history`; if stuck: `helm rollback <REL> <PREV-REV> -n <NS>` or patch the secret: `kubectl -n <NS> patch secret sh.helm.release.v1.<REL>.v<N> -p '{"metadata":{"labels":{"status":"deployed"}}}'` (last resort).
- **`pending-upgrade` / `pending-rollback`** → `helm rollback <REL> <LAST-DEPLOYED-REV> -n <NS>`
- **Hook failed** → `kubectl -n <NS> get jobs,pods -l helm.sh/hook` ; logs of the hook pod
- **CRD not found** → CRDs aren't templated on upgrade; install separately: `kubectl apply -f crds/`
- **Dependency missing** → `helm dependency update <CHART>`
- **Auth to OCI/registry** → `helm registry login <REG>`

# Safe upgrade
```
helm upgrade <REL> <CHART> -n <NS> -f values.yaml \
  --version <V> --atomic --timeout 5m --history-max 10
```
`--atomic` auto-rolls-back on failure. Rollback manual: `helm rollback <REL> <REV> -n <NS> --wait`.

# Uninstall (destructive)
`helm uninstall <REL> -n <NS>` — rollback: reinstall from `helm get manifest` snapshot taken first.
