---
pr: kubernetes-sigs/prow#538
title: "tide: remove PR from batch when batch fails"
head_sha: d64d51d24470e07d29c109336264b46f93a61156
base: main
reviewed_at: 2026-07-22T15:36:43Z
verdict: approve
---

## Summary

Fixes #474. `accumulateBatch` previously discarded `failureState` batch info, so `takeAction` kept re-triggering the exact same failing batch combination forever (only broken by a human `/hold`-ing a PR). This PR adds a `failedBatch` return value to `accumulateBatch` and, in `takeAction`, uses a new `pickLowestPriorityPR` helper to exclude the highest-numbered (newest/lowest-priority) PR from the failed batch before re-triggering a smaller batch via `pickBatch`.

Reviewed via three independent perspectives (code quality, maintainability, deployment risk) plus advisor synthesis. Advisor recommendation: APPROVE_WITH_SUGGESTIONS. No critical issues found by any reviewer; all converging concerns are follow-up quality items, not merge blockers.

## Findings

### [should-fix] pickLowestPriorityPR name/semantics conflict with pickHighestPriorityPR
- where: `pkg/tide/tide.go:934-947`
- concern: Flagged independently by all three reviewer perspectives (code quality, maintainability, deployment risk) — the strongest-confidence finding in this review. `pickHighestPriorityPR` honors `config.Tide.Priority` label-tier configuration before falling back to PR number. `pickLowestPriorityPR` only ever compares `pr.Number`, ignoring configured priority tiers entirely. The shared "priority" terminology invites future readers (and operators relying on priority labels for hotfix/release-blocking PRs) to wrongly assume both respect the same config. Rename (e.g. `pickNewestPR`/`pickHighestNumberedPR`) and/or document explicitly that it is PR-number-only, not tier-aware.
- excerpt: |
    func pickLowestPriorityPR(prs []CodeReviewCommon) (bool, CodeReviewCommon) {
    	if len(prs) == 0 {
    		return false, CodeReviewCommon{}
    	}
    	result := prs[0]
    	for _, pr := range prs[1:] {
    		if pr.Number > result.Number {
    			result = pr
    		}
    	}
    	return true, result
    }

### [should-fix] Nondeterministic failedBatch selection when multiple failed refs exist
- where: `pkg/tide/tide.go:1059-1067`
- concern: Flagged by code quality and deployment risk reviewers. The assignment loop iterates a Go map (`states`), randomized order, and overwrites `failedBatch` on every matching ref. The existing comment documents this limitation for success/pending batches, but this PR's own bisection behavior makes multiple simultaneously-failed batch ProwJobs more likely (an older full-size failing batch's ProwJob can persist for hours until GC while a newly-triggered smaller batch also fails) — so a different PR could be excluded on different sync cycles purely due to map iteration order. This now drives real bisection behavior, not just a log/metrics choice.
- excerpt: |
    switch overallState {
    // Currently we only consider 1 pending batch and 1 success batch at a time.
    // If more are somehow present they will be ignored.
    case pendingState:
        pendingBatch = state.prs
    case successState:
        successBatch = state.prs
    case failureState:
        failedBatch = state.prs
    }

### [should-fix] New bisection test doesn't assert the PR was actually excluded
- where: `pkg/tide/tide_test.go:1966-2001`
- concern: Flagged by code quality reviewer. The test only checks `numCreated`, `triggeredBatches`, and that batch jobs have `len(Pulls) > 1`. It never inspects `job.Spec.Refs.Pulls` to confirm PR 2 (the intended excluded PR) is actually missing. Since `batchLimit` is unset, a batch of all 3 PRs would satisfy the same assertions — the test would still pass if the exclusion logic were deleted or broken.
- excerpt: |
    merged:           0,
    triggered:        2,
    triggeredBatches: 2,
    action:           TriggerBatch,

### [nit] accumulateBatch doc comment not updated for failedBatch
- where: `pkg/tide/tide.go:966-970`
- concern: Flagged by maintainability and code-quality-adjacent findings. The function doc comment still only documents `successBatch` and `pendingBatch`; the new third return value is undocumented.
- excerpt: |
    // accumulateBatch looks at existing batch ProwJobs and, if applicable, returns:
    // * A list of PRs that are part of a batch test that finished successfully
    // * A list of PRs that are part of a batch test that hasn't finished yet but
    // didn't have any failures so far

### [nit] takeAction's same-typed slice parameter list keeps growing
- where: `pkg/tide/tide.go:1571`
- concern: Flagged by maintainability reviewer. `takeAction` already had several adjacent `[]CodeReviewCommon` parameters (`batchPending, successes, pendings, missings, batchMerges`); this PR adds a sixth (`batchFailed`) positionally in the middle. Nothing at the call site or type system prevents accidentally swapping two of these in a future edit. Pre-existing debt, but this PR was a natural point to convert to a small struct instead.
- excerpt: |
    func (c *syncController) takeAction(sp subpool, batchPending, batchFailed, successes, pendings, missings, batchMerges []CodeReviewCommon, missingSerialTests map[int][]config.Presubmit) (Action, []CodeReviewCommon, error) {

### [nit] First new test case doesn't exercise the new bisection logic
- where: `pkg/tide/tide_test.go:1969-1979`
- concern: The test `"batch merge fails, returns MergeBatch with error (issue #474)"` sets `batchMerges` non-empty, so `takeAction` returns at the very first `if len(batchMerges) > 0` branch — before `batchFailed` is ever consulted. It documents pre-existing merge-failure behavior, not the fix, despite being labeled "(issue #474)", and largely duplicates existing merge-error test coverage elsewhere in the table.
- excerpt: |
    name:        "batch merge fails, returns MergeBatch with error (issue #474)",
    batchMerges: []int{0, 1},
    mergeErrs: map[int]error{
        0: github.UnmergablePRError("merge conflict"),
        1: github.UnmergablePRError("merge conflict"),
    },

### [nit] No direct unit tests for pickLowestPriorityPR / withoutPR
- where: `pkg/tide/tide_test.go`
- concern: Flagged by maintainability reviewer. Both new helpers are only exercised indirectly through the `takeAction` integration-style test table. `pickHighestPriorityPR` has direct scenario coverage; these new helpers do not.

### [question] Bisection convergence across sync cycles
- where: `pkg/tide/tide.go:1594-1605`
- concern: Once PR N is excluded and the smaller batch succeeds, does the stale failed ProwJob (still matching valid pulls) keep causing `accumulateBatch` to return the old failed batch on subsequent cycles, permanently excluding PR N? Or does the new `successBatch` take precedence once it appears?
- excerpt: |
    if len(sp.prs) > 1 && len(batchPending) == 0 {
        batchSP := sp
        if len(batchFailed) > 0 {
            if ok, pr := pickLowestPriorityPR(batchFailed); ok {
                sp.log.WithField("excluded_pr", pr.Number).Info(
                    "Bisecting failed batch by excluding lowest-priority PR")
                batchSP.prs = withoutPR(sp.prs, pr.Number)
            }
        }

### [question] Behavioral shift for operators relying on manual /hold recovery
- where: n/a (deployment risk)
- concern: Deployment risk reviewer flagged that Tide now auto-retries a smaller batch on every batch CI failure with no opt-out/feature-gate, whereas previously a failed batch fell through to `MergeBatch` regardless of error (the bug) and some operators may have built manual `/hold`-based runbooks around unsticking it. Worth a release-note mention; not a merge blocker since the new behavior is strictly better and safely revertible (no persisted state incompatibility).

## Checked
- `withoutPR` allocates a new slice and does not mutate its input.
- `batchSP := sp` is a struct copy; reassigning `batchSP.prs` does not affect `sp.prs` (safe today, though a shallow copy could become a hazard if `subpool` gains pointer/map fields later).
- `pickLowestPriorityPR`'s "highest number = lowest priority" assumption matches `pickHighestPriorityPR`'s convention (smallest number = highest priority / FIFO) and `pickBatch`'s own oldest-first ordering.
- The `if len(batch) > 1` guard downstream correctly prevents triggering a degenerate single-PR "batch" if bisection reduces the pool below 2 candidates.
- No security-relevant changes; purely internal control-flow/data structure changes, no config/API/CRD/RBAC changes.
- Behavior is unchanged when no batch has failed (backward compatible, zero upgrade friction); rollback is safe.
- Bisection is self-terminating (monotonically shrinks the batch), with defensive empty-input handling in both new helpers.

## Open questions
- Should `pickLowestPriorityPR` respect `config.Tide.Priority` labels like `pickHighestPriorityPR` does, or is ignoring them intentional for bisection? If intentional, a comment explaining why would help.
- Can multiple failed-batch ProwJobs coexist in practice given this PR's own behavior, and if so, should `failedBatch` selection be made deterministic (e.g. most recent StartTime, or largest) instead of relying on map iteration order?
- Does the exclusion of PR N eventually get "forgotten" once the smaller batch succeeds, or can a stale failed ProwJob keep PR N excluded indefinitely?
