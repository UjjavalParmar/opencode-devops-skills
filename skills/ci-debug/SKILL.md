---
name: ci-debug
description: Debug CI/CD pipeline failures — GitHub Actions, GitLab CI, ArgoCD-driven CD. Covers flaky jobs, cache misses, OIDC/cloud auth failures, runner exhaustion, artifact issues, secret scoping. For ArgoCD app sync failures use argocd-debug.
---

# Triage order
1. **Read the failing step's last 50 lines, not the whole log.**
2. Compare with last green run — same SHA? same runner? same image tag?
3. Re-run with debug:
   - GHA: re-run with `ACTIONS_STEP_DEBUG=true` / `ACTIONS_RUNNER_DEBUG=true` as repo secrets
   - GitLab: `CI_DEBUG_TRACE=true` (do NOT enable on jobs that print secrets)
4. Reproduce locally with the same container image the runner used.

# GitHub Actions
```
gh run list -w <WF> -L 5
gh run view <RUN-ID> --log-failed
gh api repos/<O>/<R>/actions/runs/<ID>/timing
```
Common:
- **OIDC to AWS fails** → trust policy `sub` claim mismatch (`repo:<O>/<R>:ref:refs/heads/<B>`); `aud=sts.amazonaws.com`
- **Cache miss on every run** → cache key contains unstable input (timestamp, `${{ github.run_id }}`)
- **Permissions error** → workflow `permissions:` block too narrow (e.g., missing `id-token: write`)
- **Hosted runner OOM** → switch to larger runner or self-hosted; jobs >7GB need it

# GitLab CI
```
glab ci view <PIPELINE-ID>
glab ci trace <JOB-ID>
```
- **`fatal: unable to access`** → `CI_JOB_TOKEN` scope; project access tokens
- **Cache not shared** → `key:` per branch by default; use `key: files: [...]`
- **Runner stuck** → `gitlab-runner verify` ; check concurrent limit

# Cross-cutting
- **Flake** → quarantine via `continue-on-error` only with an issue tracking it; do not ignore silently
- **Slow** → matrix parallelism, dependency cache, split test shards
- **Secret leaked in log** → rotate immediately; force-purge log only after rotation

# Safe re-run pattern
- Re-run failed jobs only (`gh run rerun <ID> --failed`) to keep evidence.
