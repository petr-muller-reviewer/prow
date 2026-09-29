---
issue: kubernetes-sigs/prow#673
title: "Tide gets stuck retrying unmergeable PR instead of advancing to next candidate"
state: closed
labels: ""
main_sha: cca1df41e31da63d6f8d3bb790721d172b7f7843
triaged_at: 2026-09-29T17:59:20Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 1
recommended_labels: [kind/bug, area/tide]
---

# Triage

## Verdict

**Accepted, resolved for the reported deployment; leave closed.**

This was a legitimate Tide merge-pool starvation report. With the default `github_merge_blocks_policy: permit`, a PR blocked by GitHub can remain eligible and repeatedly win Tide's candidate selection. On 2026-08-04, @Prucek recommended the existing `block` policy from merged PR #579; @kaovilai confirmed that configuring it solved the reported problem and closed #673. The matching OADP configuration change merged in openshift/release#82902. Open PR #674 proposes broader handling of merge failures and merits a separate decision.

## What the issue reports

- A PR has Tide's required labels and passing checks, but lacks enough GitHub approving reviews under branch protection with `enforce_admins: true`.
- GitHub rejects Tide's merge attempt. On the next sync, Tide can select that PR again while another ready PR waits.
- @tuminoid also observed a retry loop when a PR has a changes-requested review despite `lgtm` and approval labels.
- Since the prior triage, @Prucek identified `github_merge_blocks_policy: block` as a remedy. @kaovilai confirmed it worked and closed #673 on 2026-08-04.

## Findings

### [reproducibility] Review protection can starve the merge pool
- detail: The reported conditions are specific and actionable. The code path supports repeated selection when GitHub rejects a candidate, though I did not run an end-to-end reproduction.
- evidence: kubernetes-sigs/prow#673 reproduction steps; @tuminoid's 2026-04-02 comment.

### [cause] Default permit policy leaves GitHub-blocked PRs eligible
- detail: The default policy allows a GitHub `BLOCKED` PR into the pool. `accumulate` uses presubmit results without merge-failure memory, and `pickHighestPriorityPR` selects the lowest-numbered eligible PR in a priority tier. A failed individual merge does not exclude it from the next sync.
- evidence: `pkg/config/tide.go:399-413`; `pkg/tide/tide.go:923-945,1089-1148,1556-1575`; `pkg/tide/github.go:286-319`.

### [related-code] GitHub merge-block filter
- where: `pkg/tide/github.go:636-647`; `pkg/tide/tide.go:583-588,741-775`
- excerpt: |
    if pr.MergeStateStatus == MergeStateStatusBlocked {
        switch policy := m.config().Tide.GitHubMergeBlocksPolicy(orgRepo); policy {
        case config.GitHubMergeBlocksBlock:
            return "PR is blocked from merging by GitHub (check branch protection, required reviews, or rulesets)", nil

### [related-code] Unmergeable error only advances within a batch
- where: `pkg/tide/tide.go:1488-1490`; `pkg/tide/github.go:286-319`
- excerpt: |
    } else if _, ok = err.(github.UnmergablePRError); ok {
        return true, fmt.Errorf("PR is unmergable. Do the Tide merge requirements match the GitHub settings for the repo? %w", err)

### [related-issue] #134 — reviewer-count enforcement
- ref: kubernetes-sigs/prow#134
- relevance: Related branch-protection behavior, but not the same retry-loop report; open.

### [related-issue] #269 — changes-requested retry loop
- ref: kubernetes-sigs/prow#269
- relevance: Older open report of repeated merge attempts after a changes-requested review; overlaps the failure mode but not #673's review-count scenario.

### [related-pr] #579 — configurable GitHub merge-block policy
- ref: kubernetes-sigs/prow#579
- relevance: Merged 2026-05-31. Its `block` setting was confirmed by the author as the resolution for #673.

### [related-pr] #674 — temporary exclusion after merge rejection
- ref: kubernetes-sigs/prow#674
- relevance: Open, unmerged broader fix. The current diff tracks PR/head-SHA exclusions for three sync cycles and covers individual and batch merge errors; it should be assessed separately from this resolved issue.

### [related-pr] openshift/release#82902 — OADP policy configuration
- ref: openshift/release#82902
- relevance: Merged 2026-08-04; configures `github_merge_blocks_policy: block` for 21 OADP ecosystem repositories.

## Resolved

- **Reported OADP stall:** @kaovilai wrote "Solved by configuring option added via #579" on 2026-08-04T17:07:46Z and closed #673. The matching OADP configuration PR merged shortly beforehand. This confirms the reported deployment's resolution; it does not prove every `UnmergablePRError` scenario is covered by `block`.

## Checked

- Read #673, its comments and cross-reference timeline; confirmed closure at 2026-08-04T17:07:46Z and no labels.
- Searched `tide unmergeable` and `tide merge blocked`; inspected #134 and #269. Neither is an exact duplicate of the reported combination.
- Confirmed #579 and openshift/release#82902 are merged and #674 remains open. Read #674's diff.
- Read merge filtering, candidate selection, error handling, and policy lookup at upstream main `cca1df41e31da63d6f8d3bb790721d172b7f7843`.
- Existing tests cover `BLOCKED` under `block`, `permit`, and the default (`pkg/tide/tide_test.go:1527-1570`), plus continuing after a batch merge error (`pkg/tide/tide_test.go:2079-2088`). I found no merged cross-sync advancement test after an individual merge failure.
- Completed the maintainer briefing on 2026-09-29. The confirmed configuration remedy is Level 1; generalized Tide merge-path changes such as #674 are Level 3. No maintainer decision beyond proceeding through the briefing was recorded.

## Next steps

- Leave #673 closed; the reporter confirmed resolution and no further change is needed on this issue.
- Assess #674 independently if maintainers want retry protection for deployments using `permit` or for rejection causes that do not surface as GitHub `BLOCKED`.
- If work on #674 proceeds, review batch, status, and retry semantics with Tide maintainers. No new labels are needed on the closed #673.

## Open questions

- Does #674 address a still-relevant failure mode for deployments that intentionally use the default `permit` policy?
