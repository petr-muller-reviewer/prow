---
issue: kubernetes-sigs/prow#673
title: "Tide gets stuck retrying unmergeable PR instead of advancing to next candidate"
state: open
labels: ""
main_sha: ae8e2d87967f0a2b45cfed2c514f5ec91b964596
triaged_at: 2026-07-26T23:32:43Z
verdict: accepted
refresh_log:
  - previous_triaged_at: 2026-07-01T12:47:15Z
    summary: "lifecycle/stale removed by @kaovilai via /remove-lifecycle stale comment; no other activity. PR #674 still open, not merged."
---

## Findings

### [reproducibility] Reliably reproducible merge queue stall
- detail: When branch protection has `enforce_admins: true` and `required_approving_review_count: N`, a PR with lgtm+approved labels but fewer than N GitHub reviews causes Tide to retry the same PR every sync cycle, blocking all other PRs in the pool indefinitely.
- evidence: Reproduction steps in issue body. Observed across 23+ repositories in OADP ecosystem (https://github.com/oadp-rebasebot/oadp-rebase/pull/16). Member @tuminoid confirmed the same loop with "changes requested" review verdicts.

### [cause] accumulate() has no merge failure memory
- detail: `accumulate()` classifies PRs into successes/pendings/missings based solely on ProwJob/CI status. It has no awareness of prior merge failures, so an unmergeable PR with passing CI re-enters `successes` every sync cycle.
- evidence: `pkg/tide/tide.go:1077-1135`

### [cause] tryMerge() does not prevent cross-cycle re-selection
- detail: `tryMerge()` returns `keepTrying=true` for `UnmergablePRError`, which means "continue to next PR in current batch." For single-PR merges this just returns. Nothing records the failure for future cycles.
- evidence: `pkg/tide/tide.go:1474-1475` — `return true, fmt.Errorf("PR is unmergable. Do the Tide merge requirements match the GitHub settings for the repo? %w", err)`

### [cause] Deterministic PR selection amplifies the problem
- detail: `pickHighestPriorityPR()` deterministically selects the lowest-numbered PR from `successes`. The same unmergeable PR wins every cycle, starving all other ready-to-merge PRs.
- evidence: `pkg/tide/tide.go:912-935`

### [related-code] takeAction() — merge candidate selection
- where: `pkg/tide/tide.go:1542-1585`
- excerpt: |
    if ok, pr := pickHighestPriorityPR(sp.log, successes, sp.cc, c.isPassingTests, c.config().Tide.Priority); ok {
        merged, err = c.provider.mergePRs(sp, []CodeReviewCommon{pr}, c.statusUpdate.dontUpdateStatus)
        return Merge, []CodeReviewCommon{pr}, err
    }

### [related-code] mergePRs() — merge attempt and error handling
- where: `pkg/tide/github.go:243-314`
- excerpt: |
    keepTrying, err := tryMerge(func() error {
        ghMergeDetails := gi.prepareMergeDetails(commitTemplates, pr, *mergeMethod)
        return gi.ghc.Merge(sp.org, sp.repo, pr.Number, ghMergeDetails)
    })
    if err != nil {
        log.WithError(err).Debug("Merge failed.")

### [related-code] isAllowedToMerge() — partial mitigation via MergeStateStatus
- where: `pkg/tide/github.go:600-638`
- excerpt: |
    if pr.MergeStateStatus == MergeStateStatusBlocked {
        switch policy := m.config().Tide.GitHubMergeBlocksPolicy(orgRepo); policy {
        case config.GitHubMergeBlocksBlock:
            return "PR is blocked from merging by GitHub ...", nil
        case config.GitHubMergeBlocksPermit:
            // Allow merge but the warning will be surfaced in PR status

### [related-code] GitHubMergeBlocksPolicy default is "permit"
- where: `pkg/config/tide.go:399-413`
- excerpt: |
    return GitHubMergeBlocksPermit

### [related-code] UnmergablePRError type definition
- where: `pkg/github/client.go:4211-4214`
- excerpt: |
    type UnmergablePRError string
    func (e UnmergablePRError) Error() string { return string(e) }

### [related-issue] #134 — Tide not honoring multiple reviewer branch protection
- ref: kubernetes-sigs/prow#134
- relevance: Complementary failure mode. Without `enforce_admins`, Tide merges with 0 reviews (bypasses review count). With `enforce_admins`, Tide is correctly blocked but enters the retry loop described in #673.

### [related-pr] #674 — Temporary exclusion mechanism for unmergeable PRs
- ref: kubernetes-sigs/prow#674
- relevance: Direct fix by the same author. Adds `mergeExclusions` map to `syncController` with 5-minute TTL and SHA-scoped invalidation. Filters excluded PRs in `takeAction()` before `pickHighestPriorityPR()`. Includes tests for skip behavior, error detection, TTL expiry, SHA invalidation.

### [related-pr] #538 — Remove PR from batch when batch fails
- ref: kubernetes-sigs/prow#538
- relevance: Related batch-level merge failure handling. Open.

## Checked

- All code references from issue body verified against HEAD ae8e2d87
- `syncController` struct (`pkg/tide/tide.go:79-98`) has no merge failure tracking fields
- `accumulate()` has no merge history awareness — purely ProwJob status
- `MergeStateStatusBlocked` / `github_merge_blocks_policy` checked as partial mitigation — default `"permit"` does not prevent the issue
- Gerrit provider (`pkg/tide/gerrit.go`) treats all merge errors as immediate failures with no retry — not affected
- No tests verify cross-cycle advancement past stuck/unmergeable PRs
- @tuminoid member comment confirms same loop with "changes requested" review verdicts + lgtm/approve labels

## Next steps

- Review PR #674 — directly addresses this issue with a well-scoped, tested fix
- `lifecycle/stale` already removed (by @kaovilai, 2026-07-01); still apply `kind/bug`, `area/tide`
- Verify @tuminoid's "changes requested" scenario is covered by PR #674 (any `UnmergablePRError` triggers exclusion regardless of specific rejection reason)
- Consider as follow-up whether `github_merge_blocks_policy` default should change from `"permit"` to `"block"`

Since previous triage: @kaovilai removed the `lifecycle/stale` label via `/remove-lifecycle stale` on 2026-07-01. No other activity; PR #674 remains open and unmerged.

## Open questions

- Should PR #674 be adopted as-is, or does the maintainer team prefer a different approach?
- Is 5 minutes the right TTL for the merge exclusion? Should it be configurable via Tide config?
- Should the `github_merge_blocks_policy` default be reconsidered as a related follow-up?
