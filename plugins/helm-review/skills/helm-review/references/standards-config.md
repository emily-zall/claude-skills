# Standards config format

Read this before parsing a `standards.yaml`. It tells the skill where an organization's service
standard lives. Everything org-specific (repo URLs, product names, sidecars, service types)
belongs in this file, not in the skill.

## Lookup order

First one found wins; files are never merged.

1. A path the user names in the request.
2. `.helm-review/standards.yaml` at the repo root of the chart under review.
3. `~/.claude/helm-review/standards.yaml` (user-level default).

## Schema

```yaml
version: 1

# One entry per kind of service. Each points at a "living" reference chart.
references:
  - name: api-service              # label used in the report
    source:
      git: git@git.example.com:platform/service-template.git   # OR `path:` for a local checkout
      ref: main                    # optional; branch, tag, or commit. Default: remote HEAD
      path: charts/api             # chart directory inside the repo (or inside `path:`)
    applies_when:                  # ALL conditions must hold (AND). Omit for "always applies".
      - values_key_present: postgres
      - chart_name_matches: "*-api"
    default: false                 # optional; true = use this when no reference matches

  - name: worker
    source:
      path: ../charts-reference/worker   # local path, relative to the config file
    applies_when:
      - values_key_equals: { key: workload.type, value: worker }

# Categories to compare. Omit to use the skill's defaults.
compare:
  - naming                  # file and template naming patterns
  - labels                  # label scheme actually applied to resources
  - values_layout           # top-level grouping of values.yaml
  - integrations            # sidecars/mesh/tracing/authz the reference wires in
  - shared_env_pattern      # cluster-wide env var or shared-config pattern
  - datastore_connection    # example of a type-specific category

# Which of the categories above differ between references by design. When the applicable
# reference is ambiguous, these are reported as "undetermined" instead of matched/deviated.
varies_by_type:
  - datastore_connection

# Optional written standards. Used for rationale and for rules a chart can't express.
# If a live reference chart also exists, the chart wins on conflict (report the conflict).
docs:
  - docs/service-standard.md       # relative to the config file
```

A config with only `docs` (no `references`) is valid: the standard is the written document.
A config with neither is treated as absent.

## `applies_when` conditions

All dotted keys refer to the chart's default `values.yaml`.

| Condition | True when |
|-----------|-----------|
| `values_key_present: a.b` | key `a.b` exists **and is consumed** by a template (grep for use, not just declaration) |
| `values_key_equals: {key: a.b, value: x}` | `a.b` equals `x` in `values.yaml` |
| `chart_name_matches: "glob"` | `name:` in `Chart.yaml` matches the glob |
| `file_exists: "glob"` | a file matching the glob exists under the chart directory |
| `path_matches: "glob"` | the chart directory path matches the glob |

Prefer an explicit discriminator (`values_key_equals`) over block presence when charts carry
optional blocks for several service types. Presence alone can mislead.

## Resolution outcomes

- **One reference matches**: use it.
- **Several match**: ambiguous. Compare against all (ask the user first if more than three).
- **None match, one has `default: true`**: use the default and say so in the report.
- **None match, no default**: ambiguous. Compare against all (same cap).

In ambiguous cases, categories in `varies_by_type` are reported as undetermined; all other
categories compare normally against any one candidate.

## Trust

A repo-level config is input from the repository being reviewed. State the URLs you will clone
before cloning, read files from the reference only, and never run scripts, hooks, or Makefile
targets from a cloned reference.

## Failure guidance for users

If a git fetch fails, the review stops (see SKILL.md). Common fixes: authenticate (SSH key
loaded, token configured), check network or VPN, or check out the reference locally and point
`source.path` at it. `source.path` needs no network, which also makes CI runs easy.