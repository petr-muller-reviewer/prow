---
pr: kubernetes-sigs/prow#847
title: "deck: drop stale Prow-managed contexts with no matching ProwJob"
head_sha: 41980fbc0d3985c7f9ecc7045c73ea5e6b9b162d
base: main
reviewed_at: 2026-08-13T11:40:24Z
verdict: request-changes
---

## What this PR does

- Changes `getFullPRContext` (`cmd/deck/static/pr/pr.ts`) to delete GitHub status contexts that have no matching live `ProwJob` and whose name matches a hardcoded Prow naming convention (`pull-`, `branch-ci-`, `ci-` prefixes).
- Intent: retract stale contexts left behind by renamed/removed Prow jobs, since GitHub never retracts a status context on its own.
- Explicitly scoped to name-pattern matching rather than a real config diff; author acknowledges this won't catch non-conventionally-named renamed jobs.
- Fixes #824.

## Findings

### [blocking] Pruning deletes contexts whose ProwJob was merely garbage-collected, not actually stale
- where: `cmd/deck/static/pr/pr.ts:382-390`
- concern: `builds`/`buildContextNames` only reflects currently-live ProwJob CRs. Sinker garbage-collects ProwJob CRs after `MaxProwJobAge` (default 7 days, `pkg/config/config.go:2692`). A PR open (or viewed) longer than that will have its presubmit's ProwJob CR GC'd even though the job is still configured and the GitHub context is still accurate. This logic conflates "no ProwJob" with "job renamed/removed" and will silently delete valid, still-correct contexts from the returned list.
- excerpt: |
    const prowJobNamePattern = /^(pull-|branch-ci-|ci-)/;
    for (const [name] of contextMap) {
      if (!buildContextNames.has(name) && prowJobNamePattern.test(name)) {
        contextMap.delete(name);
      }
    }

### [blocking] Naming-convention regex can match and hide legitimate non-Prow status contexts
- where: `cmd/deck/static/pr/pr.ts:385`
- concern: `/^(pull-|branch-ci-|ci-)/` is not a guaranteed-unique Prow convention. A third-party/bot GitHub status context that happens to start with one of these prefixes but is never reported via `builds` will be deleted every single run, permanently hiding a valid, currently-reporting check from the PR status view — the opposite of the PR's own stated goal of not touching non-Prow checks.
- excerpt: |
    const prowJobNamePattern = /^(pull-|branch-ci-|ci-)/;

### [should-fix] Duplicates and conflicts with the existing server-side stale-context retirement mechanism
- where: `cmd/deck/static/pr/pr.ts:382-390`
- concern: `pkg/statusreconciler/controller.go` (`removedPresubmits` / `retireRemovedContexts`, ~lines 194 and 408) already retires GitHub statuses for presubmits actually removed from config, using real config diffs rather than a name heuristic. This client-side addition re-solves the same problem with a weaker mechanism that both misses cases (non-conventional names) and wrongly targets cases the reconciler wouldn't (GC'd-but-still-valid jobs, see above) — leaving two divergent, sometimes-contradicting definitions of "stale" in the codebase.

### [should-fix] No test coverage for the new deletion behavior
- where: `cmd/deck/static/pr/pr.ts:335`
- concern: `getFullPRContext` is newly exported with new map-deletion logic and no accompanying unit test. The only functional change in the PR ships untested, so a future edit to the regex, `buildContextNames`, or the delete-while-iterating pattern has nothing to catch a regression.

### [nit] Doc comment no longer describes the function's behavior
- where: `cmd/deck/static/pr/pr.ts:330-334`
- concern: the comment still says the function "takes all pr contexts and only replaces contexts that have existing Prow Jobs" — it doesn't mention that contexts can now be deleted outright. A reader relying on the doc to assume the output always contains every GitHub-reported context will be surprised.

## Checked
- Iteration-while-deleting on the `Map` (`for (const [name] of contextMap) { ... contextMap.delete(name) }`) is safe in JS/TS — `Map` iterators tolerate deletion of the current key during iteration.
- Tide context exclusion (`context.Context === "tide"`) is unaffected by the new logic.
- Discrepancy-detection logic for state mismatches (lines 363-380) is unchanged and unaffected by the new pruning block.

## Open questions
- How does this interact with Sinker's `MaxProwJobAge` GC of ProwJob CRs — has this been tested on a PR open longer than that retention window?
- Is the naming-convention regex meant to be configurable per-deployment, since prefixes like `ci-` are common outside Prow too?
- Given `statusreconciler` already retires removed-presubmit contexts server-side, why implement a second, weaker heuristic client-side instead of extending/reusing that mechanism?
