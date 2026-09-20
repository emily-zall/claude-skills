# Helm & Kubernetes Review Plugin

Adds a `helm-review` skill that reviews Helm charts and Kubernetes manifests for syntax/schema
correctness, adherence to this org's conventions, and human readability/maintainability — not
security posture or spec-compliance in the RBAC/secrets/deprecated-API sense. A companion
`helm-review-update-standards` skill keeps the maintainer-facing docs on those conventions
current.

## What this plugin does

The `helm-review` skill activates when you ask Claude to review, lint, clean up, or assess a Helm
chart, `values.yaml`, `_helpers.tpl`, or raw Kubernetes manifests. It checks, in order, whether the
chart renders/validates, whether it matches the org's actual conventions (compared live against a
reference chart — `proj-service` for Postgres-based services, `dpp-service` for Mongo-based ones,
re-fetched at review time rather than assumed from memory), and whether it's readable. It produces
a structured markdown report grouped by category with `file:line` findings — triaging which of
them it's confident enough to hand off as auto-applied mechanical fixes (renames, comments,
reformatting) versus which need you to fix manually or review closely, and why — then offers to
apply only the former, leaving structural, logic, or standards-setting decisions as
recommendations for you to decide on.

**Triggers include:** "is this chart readable", "does this follow our conventions", "clean up this
values.yaml", "review my helm templates", "is this valid", "readability best practices" in a
Helm/K8s context.

The `helm-review-update-standards` skill re-clones both reference charts and refreshes the
maintainer-facing docs under `docs/` — these exist so a human can see what `helm-review`'s
standards pass currently assumes the org convention is without cloning either reference repo;
`helm-review` itself never reads them, and doesn't fall back to them if the live clone fails —
standards compliance is a required pass, so a failed clone stops the entire review rather than
producing a partial report. Triggers on things like "update the helm-review
standards" or "are the helm standards docs stale" — it's a maintenance task on those docs
themselves, not a chart review.

## Scope

This skill deliberately does **not** cover:
- Security posture, RBAC, secrets management, deprecated Flux/Kubernetes API versions — use a
  GitOps/Flux repo-audit skill for that.
- Generic cross-language diff simplification — use a `simplify`-style skill for that.

It owns: `helm lint`/render/schema-validation results; naming, file/chart structure, template
complexity, `values.yaml` organization, label/annotation consistency for human filtering,
resource/probe self-explanatoriness, documentation, and consistency across sibling resources and
sibling charts — checked against both a live-fetched reference chart (org conventions) and a
general readability checklist.

## Skill contents

| File | Purpose |
|------|---------|
| `skills/helm-review/SKILL.md` | Core workflow — syntax/schema check, standards pass (vs. live reference chart, no fallback), readability pass, report, offer to fix |
| `skills/helm-review/references/readability-checklist.md` | Full readability checklist with anti-pattern examples per category, and an explicit "what not to flag" list — read by the skill every review |
| `docs/postgres-service-standard.md` | Maintainer documentation on the Postgres reference chart (`service-template`/`proj-service`), dated and commit-stamped — for a human auditing what `helm-review`'s standards pass currently assumes; the skill itself never reads this |
| `docs/mongo-service-standard.md` | Maintainer documentation on the Mongo reference chart (`dpp-service`) — same caveat |
| `examples/` | Human-only sample reports from real test runs — not read by the skill itself |
| `skills/helm-review-update-standards/SKILL.md` | Companion skill: re-fetches both reference charts and refreshes the two maintainer docs above, reporting what changed |
