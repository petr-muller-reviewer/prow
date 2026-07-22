---
pr: 538
title: "tide: remove PR from batch when batch fails"
head_sha: d64d51d24470e07d29c109336264b46f93a61156
base: main
reviewed_at: "2026-06-16"
verdict: approve-with-suggestions
---

# PR #538: tide: remove PR from batch when batch fails

**Author:** Prucek (Peter Rucek) · Fixes [#474](https://github.com/kubernetes-sigs/prow/issues/474)
**Stats:** +89 / -9 · 2 files changed
**Labels:** area/tide, do-not-merge/hold, size/M

## Description

When a batch fails to merge, Tide previously didn't account for the possibility of CI failure — the action in `takeAction` was always `MergeBatch`, regardless of the error. This sometimes led to `/hold`-ing one of the PRs from the batch, and then it would merge.

Now, Tide excludes the lowest-priority PR from the failed batch and triggers a smaller batch, breaking the cycle.

## Maintainer Advisor Recommendation

**Verdict: APPROVE WITH SUGGESTIONS**

All three independent reviewers approved with no critical or blocking issues. The change is well-scoped, low-risk, and solves a real operational problem (infinite retry of the same failing batch). The converging concerns are all minor improvements that don't need to block the merge.

## Reviewer Verdicts

### Code Quality — APPROVE

- No critical issues found
- Correct bisection logic, proper pre-allocation, defensive empty-input handling
- Non-mutating approach via shallow copy avoids side effects
- Missing direct `TestAccumulateBatch` coverage for `failedBatch`
- Comment on line 1056 should mention `failedBatch`

### Maintainability — APPROVE (BURDEN: LOW)

- Well-scoped fix proportional to the problem
- Priority model inconsistency: `pickLowestPriorityPR` ignores label-based priority
- 2-PR batch edge case falls through to serial testing correctly but non-obviously
- Clean pure helpers, good logging, test comments reference issue #474

### Deployment Risk — RISK: LOW

- No breaking changes — no config, API, CLI, or serialization changes
- Zero upgrade friction — behavior identical when no batch has failed
- Safe to roll back at any time
- New `batch-failed` log field for observability, negligible log volume impact

## Converging Concerns (flagged by 2+ reviewers)

### Missing `TestAccumulateBatch` coverage for `failedBatch`

**Flagged by:** Code Quality, Maintainability

The `failedBatch` return value is discarded with `_`. The path is only tested indirectly through `takeAction`. Consider adding direct test coverage in a follow-up.

### `pickLowestPriorityPR` ignores label-based priority system

**Flagged by:** Maintainability, Deployment Risk

"Priority" has two different definitions in the same file. Add a clarifying comment explaining why label-based priority is intentionally not consulted for bisection.

### Comment on line 1056 should mention `failedBatch`

**Flagged by:** Code Quality, Maintainability

The existing comment about "only consider 1 pending/success batch" should be updated to cover all three return values. Trivial fix.

## Review Checklist

- [ ] **Multiple failed batches — nondeterministic selection** (MEDIUM): `accumulateBatch` overwrites `failedBatch` for each failed ref (map iteration order is random). If multiple failed batches exist with different PR sets, a different PR could be excluded each sync cycle. Should the batches be unioned, or should the most recent / largest be selected?
- [ ] **Priority model inconsistency** (MEDIUM): `pickLowestPriorityPR` uses only PR number (highest = lowest priority). `pickHighestPriorityPR` respects `config.Tide.Priority` labels. A high-numbered PR with priority labels would still be excluded. Is this intentional?
- [ ] **Bisection is linear, not binary** (LOW): Only one PR excluded per cycle. If the failing PR is the highest-priority one, all others are tried first across multiple sync cycles (each ~3 min). The comment says "bisecting" which implies binary search. Consider if this is acceptable or if the comment is misleading.
- [ ] **Shallow copy of `subpool`** (LOW): `batchSP := sp` is a shallow struct copy. Currently safe because `batchSP.prs` is replaced with a new slice via `withoutPR`. But if `subpool` gains pointer/map fields later, this could silently share mutable state.
- [ ] **No `TestAccumulateBatch` coverage for `failedBatch`** (MEDIUM): The third return value is discarded with `_` in existing tests. There should be test cases verifying that `accumulateBatch` correctly populates `failedBatch`.
- [ ] **"batch merge fails" test case naming** (INFO): The test at line 1969 tests merge failure (not CI failure), but references issue #474 which is about CI failure loops. The two test cases could be confused. Consider if naming should be clearer.
- [ ] **`batchFailed` constructed differently than other PR lists in tests** (LOW): In the test, `batchFailed` is built as `CodeReviewCommon{Number: n}` (minimal struct) while other lists use `genPulls` (which sets HeadRefOID, Commits, etc.). Verify this doesn't mask issues where the bisection logic might need fields beyond `Number`.
- [ ] **What happens when the excluded PR was the only failing one?** (MEDIUM): After excluding PR N, the new batch of remaining PRs should pass. On the next cycle, PR N is still in the pool and could be re-added to a new batch. Does the failed batch persist across cycles, or does the new successful batch clear it? Verify this doesn't create an infinite loop with PR N.

## Diff Annotations

### pkg/tide/tide.go

**Priority model** (near `pickLowestPriorityPR`): Compare with `pickHighestPriorityPR` above: it respects `config.Tide.Priority` labels. This function only uses PR number. A PR with priority labels but a high number would still be excluded first.

**Multiple failed batches** (near `accumulateBatch` return): The comment on line 1056 says "Currently we only consider 1 pending batch and 1 success batch at a time." The same applies to `failedBatch` now — if multiple refs are in `failureState`, each overwrites the previous. Since Go map iteration is random, this is nondeterministic. Is this acceptable, or should failed batches be unioned / the largest selected?

**Shallow copy + bisection convergence** (near `takeAction` bisection logic): *Shallow copy:* `batchSP := sp` is safe today because `.prs` gets a new slice. If `subpool` gains pointer/map fields, this becomes a shared-state bug. *Convergence:* After excluding PR N and the new batch succeeds, the old failed ProwJob still exists. On the next cycle, will `accumulateBatch` still return the old failed batch? If so, PR N stays excluded even after the problem is resolved. If the new success clears it (because `successBatch` takes precedence in `takeAction`), then it's fine. Verify which path dominates.

### pkg/tide/tide_test.go

**Missing test coverage** (near `TestAccumulateBatch`): The third return value (`failedBatch`) is discarded. Existing `TestAccumulateBatch` test cases should be extended (or new cases added) to verify that `failedBatch` is correctly populated when batch jobs have `failureState`.

**Test naming** (near "batch merge fails" test case): This test case covers *merge* failure (GitHub rejects the merge), not *CI* failure. Both reference issue #474 but test different failure modes. The existing Tide behavior (returning `MergeBatch` + error) is unchanged by this PR — this case just documents it.

**Test construction asymmetry** (near "batch CI fails" test case): `batchFailed` PRs are constructed as bare `CodeReviewCommon{Number: n}`, while other PR lists use `genPulls` (which sets HeadRefOID, Commits, etc.). In production, `batchFailed` comes from `accumulateBatch` which builds full `CodeReviewCommon` objects. The minimal construction is fine for `pickLowestPriorityPR` (only reads `.Number`), but verify no downstream code needs more fields.

## PR Discussion Context

**Prucek** (2026-02-13): It happened again, and now we captured the log properly. The batch [3957, 3962, 3964] keeps cycling between WAIT and TRIGGER_BATCH with "baseimage-generator-images not passing" — the same combination retriggers endlessly.

**Prucek** (2026-02-13): @petr-muller I think this is the place that is causing the issue: tide.go#L1028-L1037. The code doesn't care about the failure state. I am not sure how to handle it properly. Should I in case of batch failure, default to merging a single PR?

## Suggested PR Comment

This looks good to me. All three independent reviews came back clean with no blocking issues. The change is well-scoped, low-risk, and correctly addresses the problem of Tide infinitely retrying the same failing batch. I would appreciate a small comment addition on `pickLowestPriorityPR` clarifying why it uses PR number rather than label-based priority, and updating the comment near line 1056 to mention `failedBatch`, but neither is blocking. Approving as-is.
