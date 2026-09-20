# Inferring conventions from sibling charts

Used in **inferred mode**: no declared standard, but the repo (or a parent directory) contains
two or more other charts. The goal is to find out what *this team actually does*, then flag
where the target quietly differs.

## 1. Find the siblings

- Look for other `Chart.yaml` files: sibling directories of the target, and other charts in the
  same repo (`charts/*`, `helm/*`, `deploy/*`).
- Exclude the target itself and any charts under it (subcharts, vendored dependencies in
  `charts/` created by `helm dependency`).
- Exclude obvious third-party charts (vendored upstream charts with different conventions), and
  say you did.
- Need at least 2 siblings. With exactly 1, treat it as weak evidence and say so; with 0, use
  baseline mode instead.
- Sample at most ~5 siblings if there are many; prefer the most recently modified, since older
  charts often predate current practice. Say which you sampled.

## 2. Extract conventions

For each sibling, note:

| Dimension | What to capture |
|-----------|-----------------|
| File naming | `deployment.yaml` vs `<name>-deployment.yaml`; one resource per file or grouped |
| Labels | which label keys appear on resources; hand-rolled or via a helper |
| Helpers | names in `_helpers.tpl` and what they do |
| values.yaml | top-level key order and grouping; naming case; comment style |
| Resources and probes | whether set, where configured, named ports |
| Integrations | sidecars, annotations, or shared config that appear in most charts |
| Docs | README present and its sections; NOTES.txt |

## 3. Decide what counts as a convention

- **Convention**: appears in at least ~75% of sampled siblings (and at least 2). Report the
  ratio ("3 of 4 siblings").
- **Split**: roughly even between two approaches. Not a convention; don't report the target as
  divergent. Mention it once as "the team is split on X", since that's itself useful.
- **Unique to one**: ignore.

## 4. Compare and report

- Divergences from a convention become **Inconsistent** findings in the Readability section,
  tagged "(inferred convention)", citing the target `file:line` and one or two sibling
  `file:line` examples.
- Never call these "violations" or "non-compliance". No one wrote the rule down, and the target
  may be the one that's right.
- If the *target* is better than the siblings (for example, it uses a shared helper where
  siblings hand-roll labels), say so: the useful advice may be "the siblings should converge on
  this".
- In the Standards section, list the derived conventions used as the yardstick, how many
  siblings backed each, and how many charts you sampled. Keep it to a short list.

## 5. Suggest promoting inferred conventions

If the inferred conventions are strong and consistent, end with one line: the team could
capture them as a reference chart plus a `standards.yaml` (see `standards-config.md`), which
would upgrade future reviews to declared mode. Offer help; don't push.