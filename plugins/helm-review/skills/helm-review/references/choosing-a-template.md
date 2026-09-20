# Choosing or creating a service template

Read this when the user wants help defining a standard service chart: either because a review
ended in baseline mode and they accepted the offer, or because they asked up front ("what
should our service template look like?").

The output is not just advice. It's a **reference chart** plus a **standards config** that this
skill can review against, so the two steps compose: create a reference, point
`standards.yaml` at it, and future reviews run in declared mode.

## Contents

1. Ask before designing
2. Common service archetypes
3. One chart, several charts, or a library chart
4. What a good reference chart contains
5. Deliverables
6. Keep it alive

## 1. Ask before designing

Don't generate a template cold. Find out:

- **What kinds of services does the team run?** Stateless HTTP/gRPC APIs, queue workers,
  scheduled jobs, stateful or datastore-backed services, gateways?
- **What must every service have?** Sidecars or mesh, tracing/metrics, authn/authz hooks,
  shared env vars, image pull secrets, ingress conventions, a datastore connection pattern?
- **Who deploys it?** Plain Helm, Flux/Argo, a CI pipeline? This affects hooks, values layering,
  and how environment overrides are structured.
- **How many services exist today?** If there are already several charts, prefer *extracting* a
  template from the best existing one (see `inferring-conventions.md`) over inventing one.

Use short questions, and only ask what you can't infer from the repo.

## 2. Common service archetypes

| Archetype | Typical shape |
|-----------|---------------|
| Stateless API | Deployment, Service, optional Ingress/HTTPRoute, HPA, PDB, probes |
| Worker / consumer | Deployment with no Service, queue config, graceful shutdown settings |
| Scheduled job | CronJob (concurrency policy, history limits, backoff), no Service |
| Stateful service | StatefulSet, headless Service, PVC templates, ordered rollout notes |
| Gateway / edge | Deployment, Service, Ingress or route, TLS wiring |

Most teams need two to four of these. Recommend the *smallest set* that covers what they run.

## 3. One chart, several charts, or a library chart

- **One reference chart per archetype**: simplest to understand; each is a complete working
  example. Best when archetypes differ a lot. Recommended default.
- **One chart with a `type` switch**: less duplication, but the templates get conditional and
  hard to read. Acceptable only for small differences (for example, with or without a
  datastore block).
- **A library chart** (`type: library`) holding shared helpers (names, labels, probes,
  standard env), with thin per-service charts on top. Best for larger fleets; costs an extra
  layer of indirection. Recommend it only when 5+ services share substantial template logic.

Explain the tradeoff in a sentence or two and let the user choose.

## 4. What a good reference chart contains

Base it on `baseline-standard.md`, then add the team's must-haves. A reference chart is read
by humans copying patterns, so optimize for clarity:

- Minimal but complete: it renders and deploys a hello-world service with default values.
- Every helper and non-obvious values key is commented.
- Uses the shared label and naming helpers everywhere (never hand-rolled).
- Demonstrates each integration the team requires, switched by clearly named values.
- Ships a `README.md` explaining how to copy it, what to rename, and what not to change.
- Passes `helm lint` and schema validation with no warnings.

## 5. Deliverables

Offer to produce, in this order (stop when the user has what they need):

1. **The reference chart(s)**, in a directory the user chooses.
2. **`.helm-review/standards.yaml`** pointing at them, with sensible `applies_when` rules and
   `compare` categories. Use `examples/.yaml` as the starting shape.
3. **A short standard doc** (one page): what the reference charts are, when to use which, and
   the rules a chart can't express (for example, "every service must set resource limits").
   List it under `docs:` in the config.
4. **A review pass** of one existing chart against the new standard, as a sanity check that the
   standard is livable.

Write files only where the user agrees, and don't overwrite existing charts.

## 6. Keep it alive

The reference chart *is* the standard, so it should be the best-maintained chart the team has.
Suggest that changes to conventions land in the reference chart first, and that its location
be stable (a dedicated repo or a `templates/` directory), because `standards.yaml` points to
it and reviews re-fetch it every time.