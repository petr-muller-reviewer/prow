---
issue: kubernetes-sigs/prow#651
title: "`tide`: batch triggered containing an already-merged PR"
state: open
labels: kind/bug, lifecycle/rotten, area/tide
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:31:45Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/bug, area/tide, help-wanted]
---

## Findings

### [reproducibility] Independently confirmed by two reporters
- detail: Original reporter (Prucek) showed Azure/ARO-HCP tide-history where PR #4297 was merged manually at 9:50 and tide fired `TRIGGER_BATCH` including that PR at 9:53. A second maintainer (raelga) independently confirmed the same pattern is "quite consistent" on the same repo (PR #4488). Maintainer petr-muller confirmed via his own investigation that this happens more often than realized, starting around Nov 2025.
- evidence: issue comments 2026-03-11 (Prucek), 2026-03-13 (petr-muller), 2026-03-17 (raelga).

### [cause] No merged/closed-state check anywhere in tide's pool pipeline
- detail: The `PullRequest` GraphQL struct tide uses has no `Merged`/`MergedAt`/`State`/`Closed` field — only `Mergeable`, `MergeStateStatus`, `CanBeRebased`, contexts, commits, labels. `filterPR()` and `mergeChecker.isAllowedToMerge()` never check merge/closed state either. A search result that is stale (still `state:open` in GitHub's index right after a merge) is therefore trusted as-is and enters the pool unfiltered.
- evidence: `pkg/tide/tide.go:1990-2026` (`PullRequest` struct), `pkg/tide/tide.go:755-793` (`filterPR`), `pkg/tide/github.go:600-627` (`isAllowedToMerge`). Only "Merged" hit in `pkg/tide/*.go` is a post-merge log line, `pkg/tide/github.go:289`, not a query result.

### [cause] Batch-trigger branch is evaluated before single-retest branch
- detail: `dividePool()` recomputes each subpool's base SHA fresh every sync (`c.provider.GetRef(...)`). ProwJobs are matched into the subpool only by exact `(org, repo, branch, sha)`. A merge bumps the SHA, which simultaneously invalidates job-match state for every PR in that subpool, not just the merged one — so several PRs look "missing" at once. `takeAction()` checks `len(sp.prs) > 1 && len(batchPending) == 0` (batch) before the single-PR retrigger condition, so the batch path wins and sweeps in the stale merged PR. This matches the maintainer's own observation that a just-merged PR shows up in `TRIGGER_BATCH` but essentially never in plain `TRIGGER`.
- evidence: `pkg/tide/tide.go:1938-1967` (`dividePool`), `pkg/tide/tide.go:1974` (SHA-keyed ProwJob matching), `pkg/tide/tide.go:1080-1136` (`accumulate`), `pkg/tide/tide.go:946-1038` (`accumulateBatch`), `pkg/tide/tide.go:1545-1588` (`takeAction`).

### [related-code] `pkg/tide/history` has no per-PR merge-attempt tracking
- where: `pkg/tide/history/history.go`
- excerpt: |
    type Record struct {
        Time      time.Time
        Action    string // e.g. "TRIGGER", "TRIGGER_BATCH", "MERGE", "MERGE_BATCH"
        BaseSHA   string
        Target    []prowapi.Pull
        Err       string
        TenantIDs []string
    }
- relevance: Floated in the issue thread as a possible basis for merge-attempt/backoff tracking. It's a pure append-only audit log keyed per subpool (`org/repo:branch`), written once per sync from `syncSubpool()` (`pkg/tide/tide.go:1809-1818`), with no per-PR lookup capability today — reusing it for backoff would need new indexing, not just consumption.

### [related-code] Natural insertion point for a fix
- where: `pkg/tide/github.go:102-151` (`GitHubProvider.Query()`)
- excerpt: |
    for _, n := range results {
        // converts every search PullRequest node via CodeReviewCommonFromPullRequest
        // and inserts unconditionally into the prs map
    }
- relevance: Every raw GraphQL search node is converted here before entering the pool — the cheapest place to add a `Merged`-state guard.

## Checked
- `filterPR()`/`isAllowedToMerge()` for any merge/closed-state check — none exists.
- Whether the `PullRequest` GraphQL struct already fetches merged state reusable for a filter — it does not; would require adding a new field.
- Whether `pkg/tide/history` already tracks per-PR merge attempts as floated in the thread — it does not; pool-level audit log only.
- Whether a broad GitHub Search API regression explains the Nov 2025 frequency increase — maintainer ruled this out since vanilla k8s.io tide does not exhibit the same pattern as strongly; alternative explanation not yet found.

## Next steps
- Post a comment summarizing the confirmed root cause and recommended fix (add a `Merged`/`MergedAt` field to the `PullRequest` GraphQL struct in `pkg/tide/tide.go:1990-2026`, filter it out in `GitHubProvider.Query()`/`search()` in `pkg/tide/github.go:102-212`), inviting a contributor.
- Apply `/help-wanted`; consider `/remove-lifecycle rotten` — the issue is still active/actionable, staleness is calendar-only.
- File the merge-attempt-backoff idea (extending `pkg/tide/history`) as a separate, lower-priority follow-up issue rather than bundling it with the primary fix.

## Open questions
- Why did the frequency of this issue apparently increase starting around Nov 2025? A broad GitHub Search API regression was considered and set aside (vanilla k8s.io tide doesn't show the same pattern), but no alternative explanation has been confirmed — worth asking if anyone has more data here.
- Does the exact GraphQL query type tide uses expose a cheap `merged`/`mergedAt` field directly on the `PullRequest` node, or would the query need restructuring? Worth a quick schema check before implementation starts.
