---
issue: kubernetes-sigs/prow#846
title: "`tide` support for Github Rulesets"
state: open
labels:
main_sha: 7e14d333447e988af71131de0bd52db54cd14906
triaged_at: 2026-08-17T22:58:10Z
verdict: accepted
---

## Findings

### [cause] No Rulesets API support anywhere in Prow
- detail: Prow's GitHub client (`pkg/github/client.go`) is hand-rolled (no vendored `go-github`) and implements only the classic branch-protection REST endpoints. There is no `Ruleset` type, no client method, and no config schema for GitHub's Repository Rulesets API anywhere in the repo. A case-insensitive repo-wide grep for "ruleset" returns only 6 files, all comment/string text from an unrelated Dec 2025 change, with zero functional ruleset code or endpoint calls.
- evidence: `pkg/github/client.go:2840-2914` (`GetBranchProtection`, `UpdateBranchProtection`, `RemoveBranchProtection`, all against `/repos/{org}/{repo}/branches/{branch}/protection`; no `/repos/{owner}/{repo}/rulesets` or `/repos/{owner}/{repo}/rules/branches/{branch}` equivalents).

### [cause] Tide's required-status-context resolution is config-only, blind to any live GitHub state (classic or ruleset)
- detail: The chain `config.Config.GetTideContextPolicy` → `Config.GetBranchProtection`/`GetPolicy` is fed exclusively from Prow's own declarative `branch-protection:` YAML config tree plus statically-configured presubmit names (`BranchRequirements`) — never from a GitHub API call. The only live classic-branch-protection call in the repo is in `branchprotector`'s reconciler, which is a write-path diff unrelated to Tide's runtime pool-sync decisions. So a required check defined only via a GitHub-side ruleset (not mirrored into Prow's `branch-protection:` config) is invisible to Tide's context policy today.
- evidence: `pkg/config/tide.go:944-995` (`GetTideContextPolicy`, called from `pkg/tide/status.go:790` and `pkg/tide/tide.go:2399`); `pkg/config/branch_protection.go:456-524` (`GetBranchProtection`/`GetPolicy`); `cmd/branchprotector/protect.go:531` (the one live `GetBranchProtection` call, unrelated to Tide).

### [related-code] Tide's merge-blocking path already delegates to GitHub's computed mergeStateStatus, likely already ruleset-compatible
- where: `pkg/tide/github.go:600-641` (`isAllowedToMerge`), `pkg/config/tide.go:250-256` (`GitHubMergeBlocksPolicyMap`)
- excerpt: |
    if pr.MergeStateStatus == MergeStateStatusBlocked {
    	switch policy := m.config().Tide.GitHubMergeBlocksPolicy(orgRepo); policy {
    	case config.GitHubMergeBlocksBlock:
    		return "PR is blocked from merging by GitHub (check branch protection, required reviews, or rulesets)", nil
- detail: `GitHubMergeBlocksPolicyMap` was added in commit `2c14b702b` ("tide: add configurable GitHub merge blocks enforcement", Dec 2025, merged via PR #579). Since GitHub computes `mergeStateStatus` server-side taking rulesets into account, this path is likely already functionally ruleset-compatible for the merge-attempt decision — the "rulesets" wording nearby is documentation catching up to that change, not new ruleset-specific logic. No test currently exercises a ruleset-blocked-PR scenario specifically.

### [related-issue] kubernetes-sigs/prow#477 — linked in the issue body but is the wrong specific issue
- ref: kubernetes-sigs/prow#477
- relevance: #477 is a `branchprotector` bug about excluded branches retaining protection, not about rulesets. The real link to #846 is the maintainer's (petr-muller) 2026-08-13 comment on #477, filed the same day as #846, which explicitly states that `branchprotector` is legacy with a poor configuration surface and that future effort should go into ruleset support instead — thanking the author for filing #846. That comment, not the issue body's own cross-link, is what establishes #846 as maintainer-endorsed direction.

## Checked
- Repo-wide case-insensitive grep for "ruleset": 6 files, all comment/string-only, no functional code (`pkg/config/tide.go`, `pkg/tide/github.go`, `pkg/tide/status.go`, `pkg/tide/tide_test.go`, `pkg/tide/status_test.go`, `pkg/config/prow-config-documented.yaml`).
- Confirmed Prow does not vendor `go-github`; GitHub client is hand-rolled, so no upstream library ruleset support to adopt.
- Confirmed `cmd/branchprotector`/`pkg/config/branch_protection.go` operate purely on Prow's declarative config, not live GitHub branch-protection/ruleset state (aside from the reconciler's own protection push/diff calls).
- Read kubernetes-sigs/prow#477 in full, including the maintainer's 2026-08-13 comment supplying the missing rationale for #846.
- Grepped `pkg/tide` for `RequiredContexts`/`ContextPolicy`/`contextChecker` to confirm no GitHub branch-protection or rulesets API calls exist in that package.

## Next steps
- Ask the author (kaovilai) to clarify intended scope: full ruleset management (branchprotector-equivalent) vs. narrower "Tide should recognize ruleset-driven required checks."
- Split into (a) a small verification/test task confirming `GitHubMergeBlocksPolicyMap` correctly handles ruleset-only blocks, and (b) a larger design proposal for ruleset-aware config before any new client/schema work begins.
- Apply labels: `kind/feature`, `area/tide` (and `area/branchprotector` for cross-reference).
- Require a short design note before any new Rulesets API client/config work starts, given the maintainer's explicit concern about repeating branchprotector's configuration-surface problems.

## Open questions
- Should ruleset support be a new parallel config surface, or should `pkg/config/branch_protection.go`'s `Policy` type be extended to also be populated from rulesets?
- Does Prow need to *manage* rulesets (write path, like branchprotector does for classic protection), or only *read/respect* them for Tide's merge and context-policy decisions? The issue title suggests read-only is the immediate need.
- Should the existing `GitHubMergeBlocksPolicyMap`/`mergeStateStatus` path be explicitly tested against ruleset-blocked PRs before any new work starts, to establish how much of this issue is already solved?
