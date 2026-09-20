# Readability Checklist

For each category: what to look for, why it matters to a human reader, and what a good fix looks
like. Not every category applies to every chart — use judgment based on what the chart actually
contains.

## 1. Naming

- **Resource/release names** should say what the thing is or does. A Deployment named `worker2`
  tells a reader nothing.
- **`values.yaml` keys**: consistent casing (Helm convention is `camelCase`), no cryptic
  abbreviations (`cfg`, `x1`, `svc`) when a full word costs nothing. Boolean flags should read
  naturally in context — `ingress.enabled`, not `ingress.flag`.
- **Template loop/scope variables**: `{{- range $svc := .Values.services }}` beats
  `{{- range $x := .Values.services }}` the moment the block is more than a couple of lines,
  because every subsequent `$x.foo` forces the reader to scroll back up to remember what `$x` is.
- **`_helpers.tpl` template names**: should be prefixed consistently with the chart name, not generic names that could clash if this chart
  is ever used as a subchart.

## 2. File & chart structure

- **One primary resource kind per template file** is the default — it keeps diffs and reviews
  scoped to one resource. Multi-document files are fine for tightly coupled pairs (a Service and
  its headless twin); flag it when unrelated kinds are crammed into one file for no clear reason.
- **Consistent file naming pattern** across the chart. If most templates follow `<kind>_<role>.yaml`
  (`deployment_webserver.yaml`, `service_webserver.yaml`), a template that breaks the pattern
  (`webserver-svc.yml`) makes the chart harder to navigate by convention.
- **`_helpers.tpl` size/focus**: fine as a grab-bag while small; once it accumulates unrelated
  helper groups (labels, image pull logic, some totally separate naming scheme), consider whether
  a reader can still find things by scanning it top to bottom.
- **Blank-line separation within a template**: as a rule of thumb, flag a run of more than ~15
  lines with no blank line marking a new logical chunk — a new container, a new env-var group
  (e.g. tracing vars vs. DB-connection vars vs. app-specific vars), a new volume/mount, a new
  conditional branch, a new resource in a multi-document file. This is about chunking a long
  template into readable pieces, not a literal line-count to enforce — a tightly related 20-line
  block with no natural seam doesn't need an arbitrary break inserted, but a long unbroken run that
  mixes several unrelated concerns (the kind of env block that piles Datadog vars, DB vars, and
  Dapr vars together with no separation) is a real reading tax worth flagging. Same guideline
  applies to `values.yaml` — see "Line breaks between sections" below.

## 3. Template complexity ("cleverness")

- **Deep nesting**: more than ~3 levels of `{{- if }}` / `{{- with }}` inside one template makes it
  genuinely hard to trace which values are in scope at any given line. Prefer flattening via a
  named template with a documented contract (what it expects in `.`) over nesting further.
- **Minified single-line templates** that chain multiple `|` pipe operations are harder to read
  than the same logic spread across a couple of lines with consistent `{{-`/`-}}` whitespace
  control. Compactness isn't a virtue here if it costs a second read-through.
- **Magic numbers/strings embedded in templates** instead of `values.yaml` — e.g. a port hardcoded
  in three different templates instead of `.Values.service.port` in one place. This hides intent
  (a reader can't tell if the repetition is deliberate or accidental) and makes the chart harder
  to configure.
- **Copy-pasted blocks** (e.g. the same 10-line label stanza pasted into every template instead of
  `{{ include "chart.labels" . }}`) — the first time someone updates one copy and not the others,
  the chart becomes internally inconsistent in a way that's invisible until it breaks something.

## 4. `values.yaml` organization & documentation

- **Logical grouping**: keys for one component together (`webserver.*`), not interleaved with
  another component's keys just because of insertion order.
- **Line breaks between sections**: ample blank lines between logically distinct groups of keys
  (e.g. between `webserver.*` and `messageHandler.*`, or between unrelated top-level blocks) make
  a long `values.yaml` scannable — same ~15-line rule of thumb as templates (see "Blank-line
  separation within a template" under "File & chart structure" above). A wall of keys with no
  visual separation forces a reader to parse indentation alone to figure out where one component's
  config ends and another's begins.
- **Comments on non-obvious fields**: not every value needs a comment, but a reader shouldn't have
  to guess *what a field is for*, *why a particular default was chosen* (e.g. `replicaCount: 3`
  or a timeout of `45s`, if it's not an obvious round number or an obvious default), or *what
  values are actually valid* (an enum-like string, a set of accepted modes/flavors, a range with
  a non-obvious floor or ceiling) when none of that is clear from the key name alone. Encourage
  adding a comment in these cases rather than leaving the reader to reverse-engineer it from
  whatever template consumes the value.
- **Consistent depth/style**: if one section uses deeply nested structured config and a sibling
  section is flat key-value with no apparent reason for the difference, flag it — the
  inconsistency itself is what costs a reader time.
- **Documentation of the values schema**: is there a `README.md` (or `values.schema.json`)
  explaining what the important knobs do? This matters most for environment-override files (see
  Edge Cases in SKILL.md) where the override's *purpose* is easy to document and easy to omit.

## 5. Labels & annotations consistency

- Check that standard `app.kubernetes.io/name`, `app.kubernetes.io/instance`, and
  `app.kubernetes.io/component` (or equivalent) labels are present and consistent across *all*
  resources in the chart. The angle here is human/tooling readability — being able to
  `kubectl get pods -l app.kubernetes.io/component=webserver` and get what you expect — not
  whether a controller depends on them for reconciliation.
- Annotations with no obvious purpose, no comment, and no naming convention tying them to
  something documented elsewhere are confusing cruft — flag them unless the purpose is
  self-evident from the key name itself.

## 6. Resource requests/limits & probes — readability angle

This is not a "did you set resource limits" check (that belongs to a different kind of audit).
The question here is narrower: **will the next person to touch this understand why these numbers
are what they are?**

- `resources: {}` as a placeholder that hides real intent (was this deliberately left unset, or
  just never filled in?) is worth flagging as ambiguous, distinct from a documented per-environment
  override.
- Oddly specific probe timings (`initialDelaySeconds: 47`) with no comment explaining where the
  number came from are a readability smell worth a question, even if the value itself is
  reasonable — the next person who needs to tune it has no starting point.

## 7. Documentation & comments

- **`NOTES.txt`**: still the generic `helm create` boilerplate, or updated to say something true
  and useful about this specific service's post-install state?
- **Non-obvious integrations** (an `EnvoyFilter`, a Kafka pubsub wiring, a sidecar with unusual
  config): does a comment or README section explain *why* this chart needs it, or does
  understanding it require tribal knowledge only the original author has?
- **Non-obvious Helm mechanics**: assume some readers of this chart aren't Helm experts, but weigh
  that against comment bloat — a comment on every `{{- with }}` or `{{- range }}` in the chart is
  noise that trains readers to skip comments entirely. Reserve this for constructs that would
  actually stall a non-expert mid-read: scope shifts that change what `.` refers to partway
  through a block, a `range` over a map where key/value destructuring isn't obvious, recursive or
  dynamically-named `include`/`tpl` calls, or `{{-`/`-}}` chomping that's load-bearing for the
  rendered output (not just tidiness). Routine, single-purpose uses of these same constructs don't
  need a comment — the bar is "would this genuinely confuse someone who reads YAML but not Helm,"
  not "does this use Helm syntax." This is distinct from the bullet above — it's not about *why
  this resource needs what it does*, it's about *what the template syntax itself is doing*.

## 8. Consistency across similar resources

- Compare sibling templates doing similar jobs (two Deployments in one chart, or the same chart
  pattern reused across services in a multi-chart repo). Same label scheme? Same structure for env
  vars, probes, resource blocks? Divergence without a stated reason means a reader can't trust
  that a pattern learned in one file transfers to the next — that's a real maintenance cost, even
  when each file is individually fine.

## What NOT to flag

Keeping this skill in its lane matters as much as the checklist itself:

- Missing NetworkPolicies, missing image signature verification, RBAC scoping, or secrets
  management — that's a security audit's job.
- Deprecated `apiVersion`s or schema violations — that's a GitOps/Flux audit skill's job.
- Suggesting logic be merged/DRY'd across *unrelated* services just because it looks similar —
  that's in `simplify`'s territory when reviewing a diff; here you're assessing one chart's own
  internal clarity, not hunting for cross-service abstraction opportunities.
- Subjective style preferences with no real ambiguity (2-space vs. 4-space YAML indent) — only
  flag inconsistency *within* the same file or chart, never a global style preference with no
  evidence it's actually confusing anyone.
- A routine, single-purpose `{{- with }}`/`{{- range }}`/`include` left uncommented — don't flag
  every use of ordinary Helm syntax as a documentation gap (see "Non-obvious Helm mechanics"
  above); that's comment bloat, not readability.
