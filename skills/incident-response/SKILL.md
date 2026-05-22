---
name: incident-response
description: Active production incident playbook — triage, mitigation-before-RCA, comms cadence, rollback strategy, evidence preservation. Use when user signals "outage", "p0/p1/sev1/sev2", "users down", "alerting firing now". NOT for routine debugging.
---

# Preconditions
- Confirm cluster/account: `kubectl config current-context` ; `aws sts get-caller-identity`.
- One IC owns the channel. If unset, the agent's first message is: "Need IC name to proceed."

# Escalation
- Data-at-risk (DB write corruption, irreversible delete) → page DBA + freeze writes before any other action.
- Auth/secrets compromise → page security lead; treat as Sev1 even if user-impact low.
- >30 min Sev1 with no mitigation candidate → page next on-call tier.

# Phase 1 — Stabilize (first 5 min)
1. **Declare severity**. Sev1=user-impacting outage, Sev2=degraded.
2. **Roles**: IC (incident commander), Ops (hands on keyboard), Comms (updates). One person ≠ two roles in Sev1.
3. **Mitigate, don't diagnose**. Pick fastest revert:
   - Recent deploy? `kubectl rollout undo deploy/<D> -n <NS>` / `argocd app rollback <APP> <REV>` / `helm rollback`
   - Recent infra change? `git revert <SHA>` + `terraform apply`
   - Traffic spike? scale up, enable rate limit, shed load
   - Bad config flag? toggle flag back
4. **Freeze**: pause CI/CD merges to affected systems.

# Phase 2 — Evidence (preserve BEFORE further changes)
```
kubectl get events -A --sort-by=.lastTimestamp > evidence/events.txt
kubectl describe pod <BAD> -n <NS> > evidence/pod.txt
kubectl logs <BAD> -n <NS> --all-containers --previous > evidence/logs-prev.txt
```
Snapshot dashboards (Grafana share link with `from`/`to`). Note exact timestamps (UTC).

# Phase 3 — Comms
- Cadence: Sev1 every 15 min, Sev2 every 30 min.
- Template: `[<SEV>] <SYSTEM> — <USER IMPACT> — current action: <X> — next update: <HH:MM UTC>`
- Stop saying "investigating" without naming what.

# Phase 4 — Verify recovery
- Synthetic check passes from outside the cluster.
- Error budget burn rate back under SLO threshold.
- Hold for at least 2× the MTTD before declaring resolved.

# Phase 5 — Post-incident
- Timeline (UTC, machine-pulled where possible).
- Root cause distinct from trigger.
- Action items with owner + date; tag P0 = block other work.
- Blameless writeup. No "human error" without system-fix item.

# Anti-patterns
- Multiple parallel mitigations without coordination.
- Restarting things "just in case".
- `kubectl delete` on prod without snapshot/manifest dump.
- Long Slack threads with no IC summary.
