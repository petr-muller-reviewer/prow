---
issue: kubernetes-sigs/prow#400
title: "`tide` merge queue stalls when unresolved comments exist"
state: open
labels: kind/bug, lifecycle/stale, area/tide
main_sha: 53a81071b1f5a432c33362a42aa7bc9837f7ed14
triaged_at: 2026-06-03T11:33:36Z
verdict: accepted
---

## Findings

### [reproducibility] consistent — 405 on every merge attempt
- detail: When branch protection requires resolved conversations and a PR has unresolved ones, Tide's sync loop selects the PR, GitHub returns HTTP 405 (`UnmergablePRError`), Tide logs at debug level, and the loop repeats on the next cycle indefinitely.
- evidence: reported by aevyrie (reporter) and confirmed as same root cause as #269 by both petr-muller (2025-08-23) and aevyrie (2025-09-02).

### [cause] isAllowedToMerge does not check branch protection requirements
- detail: `isAllowedToMerge()` checks only `Mergeable == Conflicting`, valid merge method labels, and rebase capability. It does not query GitHub branch protection state. PRs blocked by "require resolved conversations" or "require approving reviews" pass the pre-merge filter and enter the infinite retry loop.
- evidence: `pkg/tide/github.go:605-636`

### [cause] stalling blocks entire subpool
- detail: `pickHighestPriorityPR()` deterministically selects the same PR each cycle. All other successful PRs in the subpool are blocked until the offending PR is manually removed or the conversation is resolved.
- evidence: `pkg/tide/tide.go` — `pickHighestPriorityPR` called each sync cycle with no per-PR failure state.

### [related-code] isAllowedToMerge filtering
- where: `pkg/tide/github.go:605-636`
- excerpt: |
    checks Mergeable == Conflicting, merge method labels, rebase capability;
    no check for mergeStateStatus or branch protection rules

### [related-code] PullRequest GraphQL query struct
- where: `pkg/tide/tide.go:~1914`
- excerpt: |
    PullRequest struct used in GraphQL query — MergeStateStatus field not present

### [related-code] CodeReviewCommon
- where: `pkg/tide/codereview.go`
- excerpt: |
    shared PR representation — would need MergeStateStatus if added to GraphQL query

### [related-issue] same root cause with "Changes Requested"
- ref: kubernetes-sigs/prow#269
- relevance: PRs with "Changes Requested" review also bypass isAllowedToMerge and stall the queue identically; same fix applies to both.

## Checked
- `isAllowedToMerge()` in `pkg/tide/github.go:605-636` — confirmed no branch protection check
- `pickHighestPriorityPR()` — no per-PR failure state or backoff
- GitHub GraphQL `mergeStateStatus` field: aggregates all branch protection rules into BLOCKED/CLEAN/BEHIND/DIRTY/DRAFT/HAS_HOOKS/UNKNOWN/UNSTABLE
- Issue comment history: two separate affected users (aevyrie, bonsairobo) have kept this alive through three stale cycles (Jul 2025, Dec 2025, Jun 2026) without maintainer action
- No cross-referenced PRs that address this issue

## Next steps
- Post draft comment (in TRIAGE.html): summarizes root cause, links #269, proposes mergeStateStatus fix, issues `/remove-lifecycle stale` and `/help-wanted`
- Apply `/remove-lifecycle stale` and `/help-wanted` bot commands
- Verify `mergeStateStatus` field is available in the existing GraphQL query context (or confirm it needs an extra field added to the query)
- Clarify one open question: does Tide set its own context to SUCCESS before the merge attempt, and would that cause mergeStateStatus to read CLEAN prematurely?

## Open questions
- Does Tide set its own required context to SUCCESS before calling merge? If so, `mergeStateStatus` might report CLEAN at query time but BLOCKED when the merge is attempted — the timing of the status update matters.
- Should BLOCKED cause a user-visible status update on the PR (via `status.go` / `requirementDiff()`) or just silent skip? Consistent with DIRTY (merge conflict) behavior would suggest a status update.
