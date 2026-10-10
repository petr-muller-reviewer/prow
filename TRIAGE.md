---
issue: kubernetes-sigs/prow#400
title: "`tide` merge queue stalls when unresolved comments exist"
state: open
labels: kind/bug, area/tide
main_sha: 53a81071b1f5a432c33362a42aa7bc9837f7ed14
triaged_at: 2026-10-10T14:45:54Z
verdict: accepted
refresh_log:
  - at: 2026-06-03T11:33:36Z
    summary: "Initial triage. lifecycle/stale removed by bonsairobo; bonsairobo has fix in progress on fork (no PR yet) using per-PR failure tracking approach."
  - at: 2026-10-10T14:45:54Z
    summary: "Since 2026-07-03: issue remains open with unchanged labels and no new comments; a new cross-reference cites downstream reports of repeated unresolved-thread merge failures and open upstream PR #674."
advice:
  advised_at: 2026-10-10T14:48:49Z
  based_on_triaged_at: 2026-10-10T14:45:54Z
---

## What the issue reports
- Tide repeatedly retries a PR that GitHub refuses to merge because unresolved review conversations are required, preventing other PRs in the subpool from progressing.
- The existing triage found the merge eligibility check does not account for that branch protection requirement and the retry loop has no per-PR backoff.

**Since previous triage:**
- There are no new comments on #400; it remains open with `kind/bug` and `area/tide` unchanged. On 2026-09-09, a cross-reference from gke-labs/kube-agents#1363 linked the issue to a larger Tide analysis, reporting 839 of 863 failed actions as an unresolved-review-thread retry loop occurring about every 85 seconds. That issue also cites open upstream PR #674, which targets the same retry behavior.

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

### [related-pr] bonsairobo's fork patch (status at original triage)
- ref: bonsairobo/prow@226a4d231faf (branch fix/tide-skip-unmergeable-prs)
- relevance: bonsairobo implemented a per-PR failure tracking fix (different approach from mergeStateStatus recommendation). Adds `recentMergeFailures` map to `mergeChecker`, keyed by `{org,repo,number}`, storing `{headSHA, timestamp}`. On `UnmergablePRError`, records the head SHA; `isAllowedToMerge` skips the PR until head SHA changes or 1h TTL expires. Returns a user-visible status reason. Files: `pkg/tide/github.go` (+~120 lines), `pkg/tide/github_test.go` (+~192 lines, 3 new test functions). Companion regression test commit: c964ac7016e1 (referenced the issue on 2026-06-04).
- note: no PR opened to kubernetes-sigs/prow as of 2026-07-03.

### [related-pr] upstream mitigation for repeated unmergeable-PR attempts
- ref: kubernetes-sigs/prow#674
- relevance: Open PR by Tiger Kaovilai, created 2026-04-01. It excludes a PR after `UnmergablePRError` for five minutes, keyed by the PR head SHA, so Tide can select other PRs; changes are in `pkg/tide/tide.go` with regression tests in `pkg/tide/tide_test.go`. The PR explicitly fixes #673 and is not linked as a fix for #400, but its described failure and mitigation match this triage's retry-loop cause and subpool-stall finding. Review whether the five-minute exclusion handles unresolved-conversation failures acceptably, including when the blocker clears without a new head SHA.

## Checked
- `isAllowedToMerge()` in `pkg/tide/github.go:605-636` — confirmed no branch protection check
- `pickHighestPriorityPR()` — no per-PR failure state or backoff
- GitHub GraphQL `mergeStateStatus` field: aggregates all branch protection rules into BLOCKED/CLEAN/BEHIND/DIRTY/DRAFT/HAS_HOOKS/UNKNOWN/UNSTABLE
- Issue comment history: two separate affected users (aevyrie, bonsairobo) have kept this alive through four stale cycles (Jul 2025, Dec 2025, Jun 2026, Jul 2026) without maintainer action
- bonsairobo's fix branch `fix/tide-skip-unmergeable-prs` in their fork: two commits (fix + regression test), approach differs from triage recommendation
- bonsairobo/prow@226a4d231faf: full patch reviewed — implementation is sound (SHA-keyed, 1h TTL, prune on cache clear, user-visible status reason)
- kubernetes-sigs/prow#674 remains open; its description and changed files confirm a five-minute, head-SHA-keyed exclusion after `UnmergablePRError`

## Next steps
- Track/review kubernetes-sigs/prow#674. Its approach matches the diagnosed retry loop and lets other PRs proceed, but verify that unresolved-conversation failures are detected and decide whether the five-minute delay after a blocker clears is acceptable.
- Compare #674's behavior with bonsairobo's fork patch, which records a user-visible reason and uses a one-hour TTL; determine whether either implementation surfaces a useful status to the affected PR.
- Since #674 is open and appears to address the failure, defer `/help-wanted` until review confirms it does not cover #400's unresolved-conversation case.

## Advice

1. **Link PR #674 from #400.** The open PR's retry suppression matches the diagnosed cause, but its body fixes #673 and the timeline for #400 has no direct reference to it. Add a comment so maintainers can track the fix against this report.

   ```bash
   gh issue comment 400 --repo kubernetes-sigs/prow --body $'> *This was generated by AI during triage.*\n\nPR #674 is open and skips a PR for five minutes after UnmergablePRError so other PRs can proceed. This appears to address the retry-loop cause reported in #400, although the PR currently fixes #673 rather than linking #400. Please track and review it for the unresolved-conversation case.'
   ```

2. **Review PR #674 before soliciting another fix.** It is still open, with mergeability `UNKNOWN` and no review decision. Check that unresolved-conversation failures trigger its exclusion and that the five-minute delay after a blocker clears is acceptable.

   ```bash
   gh pr view 674 --repo kubernetes-sigs/prow --web
   ```

   The existing `kind/bug` and `area/tide` labels are appropriate. Defer `help wanted` while this fix is in flight; no milestone is set, and the triage contains no scheduling target to assign.

## Open questions
- bonsairobo's approach: when a blocker is cleared without a new commit (e.g. reviewer dismisses "changes requested"), the PR is blocked for up to 1h by the TTL. Is that acceptable UX? The mergeStateStatus approach would unblock immediately on the next sync cycle.
- Does bonsairobo's returned reason string from `isAllowedToMerge` actually reach the PR status check visible on GitHub? Needs tracing through `status.go`/`requirementDiff()`.
- (Resolved: Tide-context-timing question from original triage does not apply to bonsairobo's approach since it does not use mergeStateStatus.)
