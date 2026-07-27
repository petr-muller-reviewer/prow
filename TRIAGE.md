---
issue: kubernetes-sigs/prow#134
title: "`tide` not honoring multiple reviewer branch protection"
state: open
labels: kind/bug, area/tide
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:35:12Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 3
recommended_labels: [kind/bug, area/tide, help-wanted]
---

## Initial validation

LEGITIMATE. Confirmed code-level bug in Tide (`pkg/tide/`), not a misconfiguration. Reporter's branch protection is correctly configured (screenshot shows GitHub UI blocking manual merge with only 1 of 2 required approvals); Tide bypasses it because it merges as a repo admin and never separately checks GitHub's review-count requirement. Already correctly labeled `kind/bug`, `area/tide` by petr-muller during the thread (2024-10-22, 2025-05-07). No relabeling needed.

## Research findings

Root cause: Tide's merge gate never consults GitHub branch protection's required-review-count. It gates purely on its own configured `tide.queries` labels (`lgtm`/`approved`) plus CI status, then makes a blind merge call. Because Tide's bot account typically has admin/bypass-branch-protection permission, and GitHub branch protection doesn't apply to admins unless `enforce_admins` ("Rules applied to everyone including administrators") is explicitly set, Tide can merge PRs a normal contributor couldn't merge through the UI.

The suggested workaround (`enforce_admins: true`) trades this bug for a different one: GitHub's Merge API then correctly rejects the doomed PR with `UnmergablePRError`, but Tide doesn't record the failure and re-selects the same PR every sync loop, starving the rest of the pool — tracked separately as issue #673, with an open non-draft fix already proposed in PR #674 (skip/cooldown for unmergeable PRs). No PR currently targets #134's root cause itself.

## Effort assessment

Level 3 (large). `pkg/tide` is ~15.9k lines, a sensitive multi-provider core component (GitHub + Gerrit via a shared `provider`/`CodeReviewCommon` abstraction). A fix needs: a new pre-merge gate parallel to `isAllowedToMerge` that fetches actual branch-protection review requirements and real GitHub review state; a config opt-in (avoid silently changing merge behavior for deployments relying on today's semantics); caching to avoid extra per-sync API cost; and sequencing with PR #674 so a review-count rejection doesn't reintroduce the #673 stuck-retry pattern. Suitable for experienced Prow contributors only — recommend `help-wanted`, not `good-first-issue`.

## Briefing summary

Briefed maintainer on 2026-07-27 across all 7 slides (issue overview, legitimacy, root cause, technical details, solution approach, effort, recommendations). No questions asked; maintainer acknowledged straight through with no deviation from the plan below.

## Findings

### [cause] Tide's merge gate never checks branch-protection review count
- detail: `mergeChecker.isAllowedToMerge` is Tide's actual pre-merge gate and only checks GraphQL `Mergeable` (conflict state) and merge-method labels — no review-count or branch-protection check exists anywhere in `pkg/tide`.
- evidence: `pkg/tide/github.go:600-619`

### [cause] Pool membership is label/CI-driven only
- detail: PR selection into the merge pool is decided entirely by `tide.queries` labels/missingLabels and Prow job/status contexts, never by GitHub's actual review state.
- evidence: `pkg/tide/tide.go:755-793` (`filterPR`), `pkg/tide/tide.go:847-858` (`filterSubpool`)

### [cause] Merge call is blind
- detail: The actual GitHub merge API call has no pre-check of required approving-review count.
- evidence: `pkg/tide/github.go:283` (`gi.ghc.Merge(...)`)

### [related-code] RequiredApprovingReviewCount exists but unused by Tide
- where: `pkg/config/branch_protection.go:96`, `pkg/github/types.go:594,649`
- excerpt: |
    Field is defined and used by branchprotector to configure GitHub branch
    protection settings, but is never read by pkg/tide for merge gating.

### [related-code] Only existing branch-protection read in Tide is unrelated
- where: `pkg/tide/tide.go:2398-2408`
- excerpt: |
    Reads branch protection config only for RequireManuallyTriggeredJobs,
    confirming no existing plumbing reads review-count settings from Tide.

### [reproducibility] Reproducible by configuration, confirmed independently twice
- detail: Any org/repo with `required_approving_review_count >= 1` and no `enforce_admins`, with a Tide bot holding admin/bypass permission, exhibits this. Confirmed by two independent reporters (dhaiducek, kaovilai) across multiple repos/orgs over ~2 years.
- evidence: issue comments 2024-04-25 (original report + screenshot), 2024-10-22 through 2026-04-01 (kaovilai, dhaiducek follow-ups)

### [related-issue] enforce_admins workaround causes a distinct retry-loop bug
- ref: kubernetes-sigs/prow#673
- relevance: Enabling `enforce_admins: true` makes GitHub correctly reject the merge, but Tide swallows the `UnmergablePRError` (`pkg/tide/github.go:280-300`, logged at Debug only) and `accumulate()`/`pickHighestPriorityPR` (`pkg/tide/tide.go` ~1012/1435) re-select the same doomed PR every sync, blocking the rest of the pool.

### [related-pr] Open fix for the #673 retry-loop
- ref: kubernetes-sigs/prow#674
- relevance: "tide: skip unmergeable PRs instead of retrying indefinitely" (author kaovilai, open, non-draft) — proposes a cooldown/exclusion cache for PRs failing with `UnmergablePRError`. Fixes #673's symptom, not #134's root cause; a #134 fix should be sequenced with this to avoid reintroducing the stuck-retry pattern.

### [related-issue] Prior-art precedent on graceful-failure expectation
- ref: kubernetes-sigs/prow#269
- relevance: Cited by petr-muller (2024-10-22) as precedent for "Tide should fail gracefully, not succeed wrongly" — same failure family (repeated merge attempts), distinct issue, not a duplicate.

## Checked

- Searched `pkg/tide`, `pkg/config`, `pkg/github` for `RequiredApprovingReviewCount`/review-count handling — confirmed absent from Tide's merge decision path.
- Verified issue #673 is open and its fix PR #674 is open/non-draft — the `enforce_admins` workaround is not currently safe to recommend without caveats.
- Verified issue #269 is open and related but not a duplicate.
- Confirmed existing labels (`kind/bug`, `area/tide`) are already correct — applied by petr-muller during the thread.

## Next steps

- Keep issue open, accepted; no author follow-up needed — report is complete, root cause already understood by the thread.
- Apply `/help-wanted` (skip `/good-first-issue` — requires experienced Prow/pkg-tide contributor).
- Optionally comment cross-linking #134, #673, and PR #674 so the full picture is in one place (currently only loosely connected via kaovilai's 2026-04-01 comment).
- If picked up for implementation: scope as an opt-in config field, add caching for the extra branch-protection lookup, and sequence with PR #674's merge.

## Open questions

- Should the fix be a hard pre-merge gate or opt-in config, given some deployments may rely on today's bypass semantics (e.g. override-command workflows)?
- Should scope include CODEOWNERS-based required reviewers, or just a plain approving-review-count check for a first PR?
- Should the check live inside `mergeChecker.isAllowedToMerge` directly, or as a separate cacheable pre-check given the added GitHub API cost per sync loop across large pools?
