---
issue: 650
title: "tide: obsolete batch ProwJobs not aborted when a new batch supersedes them"
repo: kubernetes-sigs/prow
reporter: "@Prucek"
opened: 2026-03-11
category: Feature request (reclassified from bug)
component: "Tide — pkg/tide/"
verdict: LEGITIMATE
effort: "Level 2 — Moderate"
labels:
  - area/tide
  - kind/feature
  - lifecycle/stale
triaged_at: 2026-06-09T15:13:53Z
refresh_log:
  - previous: "2026-06-09"
    summary: "k8s-triage-robot applied lifecycle/stale; help-wanted removed. No substantive new information."
---

# Triage: Issue #650

`tide`: obsolete batch ProwJobs not aborted when a new batch supersedes them

[View on GitHub](https://github.com/kubernetes-sigs/prow/issues/650)

## Overview

| Field     | Value                                                  |
|-----------|--------------------------------------------------------|
| Reporter  | [@Prucek](https://github.com/Prucek)                  |
| Opened    | 2026-03-11                                             |
| Category  | Feature request (reclassified from bug)                |
| Component | Tide — `pkg/tide/`                                     |

**Since previous triage (2026-06-09):**
- `k8s-triage-robot` applied `lifecycle/stale` at 2026-06-09T14:23:49Z (90 days of inactivity). The `help-wanted` label is no longer present.
- No substantive comments, no linked PRs, no cross-references.

When the base branch SHA advances (merge, manual or via Tide), Tide starts a new batch without aborting ProwJobs from the previous batch. The old ProwJobs continue running to completion even though their results are no longer relevant, wasting CI resources. The reporter provided a real-world example from Azure/ARO-HCP where a manual merge caused a third parallel batch to start.

## Root Cause

**In one sentence:** `dividePool()` queries ProwJobs using a cache index keyed by `org/repo:branch@baseSHA`. When baseSHA advances, old batch ProwJobs vanish from the query results — Tide can't see them, sees `batchPending == 0`, and triggers a fresh batch.

### How it happens

baseSHA advances → `dividePool()` queries new SHA → old batch ProwJobs invisible → `accumulateBatch()` returns empty → `takeAction()`: batchPending == 0 → new batch triggered

Old batch ProwJobs continue running on the cluster, orphaned. Their results will never be used.

### Contributing factors

- `TerminateOlderJobs` in `pkg/pjutil/abort.go:64` **explicitly excludes batch jobs** — Plank won't clean them up
- No state is persisted between sync iterations — Tide has no memory of previous baseSHAs
- The cache index (`cacheIndexFunc` at `tide.go:2082`) keys on baseSHA — no way to find stale-SHA jobs

## Code Map

| Function             | Location             | Role                                                        |
|----------------------|----------------------|-------------------------------------------------------------|
| `Sync()`             | `tide.go:531-623`    | Main sync loop, runs periodically                           |
| `dividePool()`       | `tide.go:1864-1909`  | Partitions PRs into subpools, queries ProwJobs by current baseSHA |
| `syncSubpool()`      | `tide.go:1721-1790`  | Syncs a single org/repo/branch subpool                      |
| `accumulateBatch()`  | `tide.go:937-1038`   | Classifies batch ProwJobs into pending/success              |
| `takeAction()`       | `tide.go:1483-1526`  | Decides action; triggers batch when `batchPending == 0`     |
| `trigger()`          | `tide.go:1425-1466`  | Creates ProwJob objects with current baseSHA                |
| `cacheIndexFunc()`   | `tide.go:2082-2095`  | Indexes ProwJobs by `org/repo:branch@baseSHA`               |

### Existing abort mechanisms

| Mechanism                    | Location                | Handles batches?                                      |
|------------------------------|-------------------------|-------------------------------------------------------|
| `pjutil.TerminateOlderJobs()`| `abort.go:58-122`      | No — explicitly excluded at line 64                   |
| `trigger.abortAllJobs()`     | `pull-request.go:170-199` | Yes, but triggered by PR events, not Tide          |
| `plank.terminateDupes()`     | `reconciler.go:444-451`| No — calls TerminateOlderJobs                         |
| `plank.syncAbortedJob()`     | `reconciler.go:783-804`| Yes — handles cleanup of any aborted job               |

### Test coverage gaps

- `TestAccumulateBatch` (`tide_test.go:89-308`) — does NOT test baseSHA change scenarios
- `TestDividePool` (`tide_test.go:933-1122`) — does NOT test what happens when baseSHA advances with pending batches
- `TestTerminateOlderJobs` (`abort_test.go:123`) — explicitly confirms batch jobs are NOT terminated

## Proposed Solutions

### Approach 1: Abort stale batches in dividePool/syncSubpool

After querying ProwJobs by current baseSHA, also query for batch ProwJobs with stale baseSHAs for the same org/repo/branch and abort them.

- **Pros:** Directly addresses the issue at the source; cleanup happens early in sync cycle
- **Cons:** Additional ProwJob query per subpool; breaks `dividePool()` read-only role

### Approach 2: Extend TerminateOlderJobs to include batch jobs

Remove the batch job exclusion in `abort.go:64` so Plank automatically aborts stale batch jobs.

- **Pros:** Minimal code change; leverages existing abort infrastructure
- **Cons:** Digest comparison may not suit batch jobs; exclusion was intentional — reason unknown

### Approach 3: Abort stale batches in takeAction before triggering (Recommended)

Before triggering a new batch, query for batch ProwJobs with stale baseSHAs for the same org/repo/branch and set them to `AbortedState`. Plank's `syncAbortedJob()` handles cleanup.

- **Pros:** Clear intent: abort old → trigger new; follows established abort pattern; no changes to `dividePool()` semantics; can be made configurable
- **Cons:** May need new cache index; only cleans up when new batch is about to start

### Key implementation considerations

1. Need a way to query batch ProwJobs by org/repo/branch *without* baseSHA — either a new index or a broader list query
2. Abort should set `Status.State = AbortedState` following the established pattern — Plank handles pod cleanup
3. Consider whether to abort on every sync or only when triggering a new batch
4. Configurability: some deployments may want stale batches to complete

## Effort Assessment

**Level 2 — Moderate (help-wanted)**

Well-defined problem with established abort patterns to follow. Moderate scope (2-4 files, ~100-200 LOC) but requires understanding Tide's batch lifecycle and ProwJob indexing.

| Factor             | Assessment                                                | Level |
|--------------------|-----------------------------------------------------------|-------|
| Scope              | 2-4 files, 100-200 LOC                                   | 2-3   |
| Complexity         | Abort pattern well-established; main challenge is index query | 2     |
| Required expertise | Tide sync loop, cache indexes, controller-runtime         | 2-3   |
| Clarity            | Well-defined problem and solution                         | 1-2   |
| Testing            | Follow existing patterns, add baseSHA-change scenarios    | 2     |
| Backwards compat   | Minor — changes behavior for stale jobs only              | 1-2   |
| Arch alignment     | Follows established abort pattern exactly                 | 1-2   |
| External deps      | None                                                      | 1-3   |

## Draft Comment

The root cause is in Tide's `dividePool()` function, which queries ProwJobs using a cache index keyed by `org/repo:branch@baseSHA`. When baseSHA advances, old batch ProwJobs no longer match the index and become invisible to subsequent sync iterations. `takeAction()` then sees `batchPending == 0` and triggers a fresh batch, while the old one continues running to completion. In effect, the old batch ProwJobs are orphaned — still consuming CI resources but with results that Tide will never use.

Prow already has a well-established pattern for aborting superseded jobs: `pjutil.TerminateOlderJobs()` in `pkg/pjutil/abort.go` does exactly this for presubmit jobs, and Plank's `syncAbortedJob()` handles the cleanup (pod deletion, marking complete). However, `TerminateOlderJobs` explicitly excludes batch jobs (line 64). The fix would involve either removing that exclusion (if batch job digest comparison works correctly for this case) or adding batch-specific abort logic in Tide itself — for example, querying for batch ProwJobs with stale baseSHAs before triggering a new batch and setting them to `AbortedState`.

`/help-wanted`

## Contributor Guide

### Files to review before starting

- `pkg/tide/tide.go` — `dividePool()`, `takeAction()`, `trigger()`, `accumulateBatch()`
- `pkg/pjutil/abort.go` — `TerminateOlderJobs()` — the established abort pattern
- `pkg/plank/reconciler.go` — `syncAbortedJob()` — how aborted jobs are cleaned up
- `pkg/tide/tide_test.go` — `TestAccumulateBatch`, `TestDividePool` — existing test patterns

### Tests to write

- baseSHA changes with pending batch → old batch aborted, new batch triggered
- Multiple stale batches from sequential baseSHA advances → all aborted
- Configuration to disable abort behavior (if config option added)
- Full sync cycle with baseSHA advance (integration-style)

### Caveats

- The `TerminateOlderJobs` batch exclusion at `abort.go:64` was intentional. Investigate before removing.
- Consider abort-on-every-sync vs. abort-only-before-new-batch. Every-sync is more aggressive but prevents orphans when no new batch is needed.
- Some deployments may want stale batches to complete (artifacts beyond merge gating). Config option could address this.
