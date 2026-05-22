# Contributing

## Adding a new skill

1. Create `skills/<skill-name>/SKILL.md`.
2. Use this frontmatter (only `name` and `description` are read by OpenCode/Claude Code):

```yaml
---
name: <skill-name>
description: <one line: what it covers AND what it does NOT cover — name the sibling skill the user should use instead>
---
```

3. Sections, in order:
   - `# Preconditions` — context check (kubeconfig, AWS profile, CLI versions). Keep to 1-3 lines. Optional for low-stakes skills.
   - `# Escalation` — when to page another team / stop. Only for skills with destructive potential.
   - `# Order of operations (read-only)` — numbered evidence-gathering commands.
   - `# Symptom → command` — table or bullet list of failure modes mapped to first command.
   - `# Mutations` — every mutating command paired with: scope, blast radius, rollback command.
   - `# Verify` — single command that proves the fix.

## Style

- Command-first. Code blocks for every command, no `$` prompts.
- Placeholders in `<UPPERCASE>`: `<NS>`, `<POD>`, `<CLUSTER>`, `<ARN>`.
- No prose explanations of what tools are. Assume advanced operator.
- Cite only: kubernetes.io, helm.sh, argo-cd.readthedocs.io, developer.hashicorp.com, docs.aws.amazon.com, docs.docker.com, docs.github.com, docs.gitlab.com, Linux man pages.
- If a flag/field's existence is uncertain, say "verify with `<command>`" — never guess.

## Anti-patterns (will be rejected)

- Overlapping scope with an existing skill without explicit exclusion in `description`.
- Mutating commands without a rollback line.
- Generic advice ("check the logs"); be specific ("`kubectl logs <POD> -n <NS> --previous --tail=200`").
- Citations to blogs, Medium, Stack Overflow, vendor marketing.
- Extended frontmatter fields (`tags`, `maturity`, `tools_allowed`, etc.) — OpenCode ignores them; they hurt retrieval.

## PR checklist

- [ ] Skill description excludes overlapping scope
- [ ] Preconditions present if skill performs mutations
- [ ] Every mutation has a rollback
- [ ] `Verify` section present
- [ ] Sources cited are official
