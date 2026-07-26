---
issue: kubernetes-sigs/prow#474
title: "2 pr's are not compatible in merge pool"
state: open
labels: kind/bug, area/tide, lifecycle/rotten
main_sha: 539ecaca1ce9f8aeeefbfd016be10d5c02876f6c
triaged_at: 2026-07-24T16:45:07Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/bug, area/tide, help-wanted]
refresh_log:
  - at: 2026-07-24T16:45:07Z
    previous_triaged_at: 2026-06-09T14:40:44Z
    summary: "No substantive activity. k8s-triage-robot auto-progressed lifecycle/stale -> lifecycle/rotten on 2026-07-24T02:59:59Z (no response to earlier stale marker). PR #538 still open and unmerged. Root cause code (accumulateBatch missing failureState case, pkg/tide/tide.go) unchanged on main; unrelated tide.go changes since (sticky overrides, prowJobsFromContexts) don't affect the finding."
---

## Findings

### [cause] accumulateBatch missing failureState case
- detail: `accumulateBatch()` switch at lines 1028-1035 handles `pendingState` and `successState` but has no `case failureState:`. Failed batches are silently discarded — never returned to `syncSubpool()` or `takeAction()`.
- evidence: `pkg/tide/tide.go:1028-1035`

### [cause] takeAction re-triggers identical failing batch
- detail: `takeAction()` checks `len(batchPending) == 0` at line 1569 to decide whether to trigger a new batch. After a failed batch is discarded by `accumulateBatch()`, this is always true, so the same batch is re-triggered every sync cycle. Step 4 (trigger batch) has higher priority than step 5 (trigger individual retest), so the individual retest path is starved.
- evidence: `pkg/tide/tide.go:1569`

### [cause] individual retest path starved by priority ordering
- detail: `takeAction` returns a single action per sync. Step 4 (trigger batch, line 1569) fires before step 5 (trigger individual retest, line 1579). Each time a failed batch clears `batchPending`, step 4 immediately re-triggers a batch and returns before step 5 is reached. Step 5 also requires `len(successes) == 0 && len(pendings) == 0`, further restricting when it can fire.
- evidence: `pkg/tide/tide.go:1542-1585`

### [related-code] accumulateBatch function
- where: `pkg/tide/tide.go:946-1038`
- excerpt: |
    switch overallState {
    // Currently we only consider 1 pending batch and 1 success batch at a time.
    // If more are somehow present they will be ignored.
    case pendingState:
        pendingBatch = state.prs
    case successState:
        successBatch = state.prs
    }
    // NO CASE FOR failureState
    return successBatch, pendingBatch

### [related-code] takeAction function
- where: `pkg/tide/tide.go:1542-1585`
- excerpt: |
    // If we have no batch, trigger one.
    if len(sp.prs) > 1 && len(batchPending) == 0 {
        batch, presubmits, err := c.pickBatch(sp, sp.cc, c.pickNewBatch)

### [related-code] syncSubpool orchestration
- where: `pkg/tide/tide.go:1778-1861`
- excerpt: |
    batchMerge, batchPending := c.accumulateBatch(sp)
    // only success and pending captured; failure silently dropped

### [related-pr] PR #538 batch bisection fix
- ref: kubernetes-sigs/prow#538
- relevance: Implements batch bisection on failure — extends `accumulateBatch()` to return `failedBatch`, adds `failureState` case, excludes lowest-priority PR from failed batch when re-triggering. Still open, authored by Prucek.

### [reproducibility] observed in production
- detail: Observed in Azure/ARO_HCP repo and openshift/dpu-operator. Occurs when two or more semantically incompatible PRs are eligible for batching, their individual test results are stale (base moved), and batch jobs fail quickly relative to the Tide sync period.
- evidence: https://github.com/kubernetes-sigs/prow/issues/474#issuecomment-4020200695

## Checked

- Commit `d64d51d24` (PR #538 fix) is NOT on main — only on `tide-batch-err` branch
- `failureState` case confirmed missing on current main at lines 1028-1035
- Related batch improvements (commits `230403b27`, `6c9724340`) also not on main
- Individual PR retest path (line 1579) requires `len(successes) == 0 && len(pendings) == 0` — not reached when PRs have any individual results
- `accumulateBatch` return signature is `(successBatch, pendingBatch)` — no third return for failures

## Since previous triage (2026-07-24)

- No new comments from the author or maintainers; the only activity was `k8s-triage-robot` auto-progressing the issue from `lifecycle/stale` to `lifecycle/rotten` on 2026-07-24T02:59:59Z, since nobody responded to the stale marker.
- PR #538 (batch bisection fix) is still open, unmerged, last updated 2026-07-15 — the fix has not landed.
- Confirmed on current `upstream/main` (past the previously-analyzed SHA): `accumulateBatch()` still has no `failureState` case; unrelated `pkg/tide/tide.go` changes since triage (sticky override support, `prowJobsFromContexts`) don't touch this code path.

## Next steps

- Review and iterate on PR #538 to get it merge-ready
- Consider whether bisection heuristic (drop lowest-priority PR) is optimal vs alternatives (binary split, drop all but highest-priority)
- `/remove-lifecycle rotten` on the issue to prevent auto-close (30 days from 2026-07-24, i.e. ~2026-08-23)
- Link PR #538 in the issue if not already done

## Open questions

- Should bisection drop the lowest-priority PR, or use a smarter convergence strategy?
- Should there be a configurable limit on batch failure cycles before falling back to individual retests?
- Are there other scenarios where `accumulateBatch` ignoring failure state causes problems beyond the infinite loop?
