---
issue: kubernetes-sigs/prow#400
title: "`tide` merge queue stalls when unresolved comments exist"
state: open
labels: kind/bug, area/tide
main_sha: 53a81071b1f5a432c33362a42aa7bc9837f7ed14
triaged_at: 2026-07-03T11:27:51Z
verdict: accepted
refresh_log:
  - at: 2026-06-03T11:33:36Z
    summary: "Initial triage. lifecycle/stale removed by bonsairobo; bonsairobo has fix in progress on fork (no PR yet) using per-PR failure tracking approach."
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

### [related-pr] fix in progress on bonsairobo's fork — no PR yet
- ref: bonsairobo/prow@226a4d231faf (branch fix/tide-skip-unmergeable-prs)
- relevance: bonsairobo implemented a per-PR failure tracking fix (different approach from mergeStateStatus recommendation). Adds `recentMergeFailures` map to `mergeChecker`, keyed by `{org,repo,number}`, storing `{headSHA, timestamp}`. On `UnmergablePRError`, records the head SHA; `isAllowedToMerge` skips the PR until head SHA changes or 1h TTL expires. Returns a user-visible status reason. Files: `pkg/tide/github.go` (+~120 lines), `pkg/tide/github_test.go` (+~192 lines, 3 new test functions). Companion regression test commit: c964ac7016e1 (referenced the issue on 2026-06-04).
- note: no PR opened to kubernetes-sigs/prow as of 2026-07-03.

## Checked
- `isAllowedToMerge()` in `pkg/tide/github.go:605-636` — confirmed no branch protection check
- `pickHighestPriorityPR()` — no per-PR failure state or backoff
- GitHub GraphQL `mergeStateStatus` field: aggregates all branch protection rules into BLOCKED/CLEAN/BEHIND/DIRTY/DRAFT/HAS_HOOKS/UNKNOWN/UNSTABLE
- Issue comment history: two separate affected users (aevyrie, bonsairobo) have kept this alive through four stale cycles (Jul 2025, Dec 2025, Jun 2026, Jul 2026) without maintainer action
- bonsairobo's fix branch `fix/tide-skip-unmergeable-prs` in their fork: two commits (fix + regression test), approach differs from triage recommendation
- bonsairobo/prow@226a4d231faf: full patch reviewed — implementation is sound (SHA-keyed, 1h TTL, prune on cache clear, user-visible status reason)

## Next steps
- `lifecycle/stale` already removed; `/help-wanted` still appropriate — post a comment inviting bonsairobo to open a PR.
- Review bonsairobo's approach against the mergeStateStatus alternative before they open a PR: per-PR failure tracking works without new GraphQL fields and avoids the Tide-context-timing open question entirely; tradeoff is a 1h window where a resolved blocker (e.g. dismissed review) still blocks the PR until TTL expires or a new commit is pushed.
- If inviting the PR: suggest they consider surfacing the skip reason via `status.go`/`requirementDiff()` for Tide status visibility (their current implementation returns the reason string from `isAllowedToMerge` but it's unclear if that surfaces to the PR status check).
- `/help-wanted` label still needs to be applied.

## Open questions
- bonsairobo's approach: when a blocker is cleared without a new commit (e.g. reviewer dismisses "changes requested"), the PR is blocked for up to 1h by the TTL. Is that acceptable UX? The mergeStateStatus approach would unblock immediately on the next sync cycle.
- Does bonsairobo's returned reason string from `isAllowedToMerge` actually reach the PR status check visible on GitHub? Needs tracing through `status.go`/`requirementDiff()`.
- (Resolved: Tide-context-timing question from original triage does not apply to bonsairobo's approach since it does not use mergeStateStatus.)
