---
pr: kubernetes-sigs/prow#959
title: "Handle concurrent job triggers"
head_sha: 5a7e1ad10941d98ad6c0099cad9c571119ba7dc7
base: main
reviewed_at: 2026-09-22T21:17:06Z
verdict: approve
---

## Verdict

Approve. The change prevents duplicate automatic presubmits for one revision, fails open if the ProwJob list cannot be read, and makes tied duplicate cleanup deterministic. It does not affect explicit `/test` or `/retest` requests.

## What this PR does

- Filters automatic PR-event presubmits whose job name already has a non-complete, non-aborted ProwJob for the same PR head and base SHA.
- Preserves CI availability by running the original set when the ProwJob list fails.
- Orders tied duplicate ProwJobs by name after start time so concurrent aborters retain the same survivor.
- Covers the trigger-side filter and all list-order permutations for equal-time duplicates.

## Findings

### [question] Consider sharing the PR-scoped ProwJob query
- where: `pkg/plugins/trigger/pull-request.go:469-491`
- concern: `runningJobsForRevision` repeats PR selector construction and listing behavior used by `abortAllJobs`. This is not a merge blocker, but a small shared query helper could prevent future selector changes from drifting.
- excerpt: |
    selector, err := labelSelectorForPR(pr)
    jobs, err := c.ProwJobClient.List(context.TODO(), metav1.ListOptions{LabelSelector: selector.String()})

### [question] Guard the automatic-versus-manual trigger policy at the entry point
- where: `pkg/plugins/trigger/pull-request.go:426-441`
- concern: The helper test verifies filtering, and the comment documents that `/test` and `/retest` bypass it. A focused entry-point test could lock in that call-path distinction if the triggering flow is refactored later.
- excerpt: |
    return RunRequested(c, pr, baseSHA, skipAlreadyRunning(c, pr, baseSHA, toTest), eventGUID)

## Checked

- `skipAlreadyRunning` matches org/repo/PR labels and verifies both head and base SHA before suppressing a job.
- ProwJob List failures fail open, so transient API errors do not drop CI.
- Completed and aborted jobs do not suppress a new run.
- `scanOrder` uses a stable name tie-breaker; the added permutation test proves the survivor is input-order independent.
- No configuration, CRD, manifest, API-contract, or RBAC changes; rolling upgrades are safe.

## Open questions

- Would a shared PR-scoped ProwJob list helper be worthwhile now, or is the current duplication preferable while the callers have different filtering needs?
- Should a future integration test exercise two simultaneous webhook handlers, beyond the acknowledged best-effort race handling?
