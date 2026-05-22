# opencode-devops-skills

Agent-ready `SKILL.md` library + `AGENTS.md` for DevOps / SRE work with [OpenCode](https://opencode.ai) and [Claude Code](https://docs.claude.com/en/docs/claude-code).

Optimized for: low token usage, command-first answers, production-safe defaults, read-only-first debugging, explicit rollback paths.

## Skills

| Skill | Use when |
|---|---|
| `kubernetes-debug` | Workload errors — CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled, RBAC, node NotReady |
| `helm-debug` | Failed install/upgrade, pending-upgrade lock, template errors, hook failures |
| `ingress-debug` | Ingress / Gateway API 404/502/503/504, TLS, ingress-nginx, AWS LB Controller |
| `dns-debug` | CoreDNS, in-cluster + host resolution, Route53 |
| `argocd-debug` | OutOfSync, Degraded, sync hooks, ApplicationSet, RBAC |
| `aws-iam-debug` | AccessDenied, AssumeRole, SCP, IRSA / Pod Identity, KMS policies |
| `eks-debug` | Node groups, Karpenter, VPC CNI, aws-auth / Access Entries, addons |
| `terraform-review` | Pre-apply review of plan / diffs / destructive changes |
| `terraform-state-recovery` | Stuck locks, corrupted/missing state, drift, accidental destroy |
| `incident-response` | Active Sev1/Sev2 — mitigation-first playbook |
| `ci-debug` | GitHub Actions, GitLab CI — failures, OIDC, caches, runners |
| `network-debug` | L3-L4 connectivity, NetworkPolicy, conntrack, MTU, SGs |
| `container-debug` | Docker/Podman build + runtime, PID1, mounts, exit codes |
| `linux-debug` | Host triage — systemd, journalctl, disk/CPU/mem/IO, FD limits |

Each skill follows: **Preconditions → Evidence → Symptom→command table → Mutations (with rollback) → Verify**.

## Install

Drop into your OpenCode / Claude Code config directory:

```bash
git clone git@github.com:UjjavalParmar/opencode-devops-skills.git ~/.config/opencode
```

Or, if you already have a config:

```bash
cd ~/.config/opencode
git clone --depth=1 git@github.com:UjjavalParmar/opencode-devops-skills.git /tmp/odv
cp -r /tmp/odv/{AGENTS.md,skills} .
```

OpenCode auto-discovers `skills/*/SKILL.md`. No further config needed.

For Claude Code, place the same files under `~/.claude/` (`AGENTS.md` → `CLAUDE.md`, `skills/` → `skills/`).

## Configuration

Use `opencode.json.example` as a starting point. Keep your real `opencode.json` out of git (the included `.gitignore` enforces this). Pass the API key via env var:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

## Design rules (see `AGENTS.md`)

- Command-first, no prose padding
- Read-only debugging before any mutation
- Every mutation paired with a rollback command
- Never invent ARNs, IDs, namespaces, flags — ask or read from evidence
- Cite only official docs (kubernetes.io, helm.sh, argo-cd.readthedocs.io, developer.hashicorp.com, docs.aws.amazon.com, docs.docker.com, man pages)

## Contributing

See `CONTRIBUTING.md`. New skills must:
1. Have a non-overlapping `description` that names exclusions (`NOT for X — use Y`)
2. Include a symptom→command table
3. Pair every mutating command with a rollback
4. Cite only official sources

## License

Apache-2.0 — see `LICENSE`.
