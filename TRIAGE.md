---
issue: kubernetes-sigs/prow#846
title: "`tide` support for Github Rulesets"
state: open
labels: kind/feature, area/tide, area/branchprotector, priority/important-longterm
main_sha: 7e14d333447e988af71131de0bd52db54cd14906
triaged_at: 2026-11-10T01:03:24Z
verdict: accepted
refresh_log:
  - at: 2026-11-10T00:56:50Z
    since: 2026-08-17T22:58:10Z
    summary: "kind/feature label now applied; petr-muller cross-referenced kubernetes-sigs/prow#624 (2026-10-10, 'Discussed in #846'), the earlier narrower ruleset-context issue that the lifecycle bot closed as not-planned; re-verified on upstream/main that no functional ruleset code exists."
  - at: 2026-11-10T01:03:24Z
    since: 2026-11-10T00:56:50Z
    summary: "petr-muller applied area/tide, area/branchprotector and priority/important-longterm and noted @hoxhaeris has looked into ruleset support; no public PR/issue/branch from hoxhaeris on rulesets, so that prior work is not in the repo."
advice:
  advised_at: 2026-11-10T00:58:09Z
  based_on_triaged_at: 2026-11-10T00:56:50Z
---

# Triage

## Verdict

Accepted — maintainer-endorsed feature request; the read-only Tide slice is the immediate, uncontroversial part, and ruleset management needs a design note first.

petr-muller explicitly endorses general Rulesets support and `/kind feature` is applied. He frames two angles: managing rulesets the way `branchprotector` manages classic branch protection, and Tide consuming that config plus whatever GitHub's API exposes for mergeability. He asks for a design document before code. The read-only ask is also independently scoped by #624 (Tide respecting required contexts from rulesets, the analogue of `tide.context_options.from-branch-protection`), which points here as canonical.

## What the issue reports

- Prow's `tide` only understands "classic" branch-protection rules; it does not support GitHub Rulesets.
- The author wants the work to also fix kubernetes-sigs/prow#477 (excluded branches retaining branch protection under `branchprotector`).
- petr-muller (2026-08-13) broadened it: Prow has no Rulesets support at all, so a first-class ruleset management concept (peribolos-like, or new) should exist with a dedicated config surface, inferring required ProwJobs from `always_run`/`optional`; Tide should consume that config and GitHub's mergeability API surface.
- Maintainer condition: start with at least a vague design doc and discussion, not code.

Since previous triage:
- 2026-10-10: petr-muller cross-referenced kubernetes-sigs/prow#624 ("`tide` does not respect required contexts from Github Rulesets") with the comment "Discussed in #846", confirming #846 is the canonical issue and that the read-only `from-branch-protection` analogue for rulesets belongs to it.
- kubernetes-sigs/prow#624 was auto-closed `NOT_PLANNED` by `k8s-triage-robot` on 2026-09-09 via the `lifecycle/rotten` flow (not a maintainer judgement), so its closure argues nothing about the feature.
- `kind/feature` is now applied (frontmatter previously recorded no labels).
- No code change: `upstream/main` advanced 364 commits since `main_sha`, but has no functional ruleset support yet.

Since previous triage (2026-11-10):
- petr-muller applied `area/tide` and `area/branchprotector` via `/area` and `priority/important-longterm` via `/priority` (01:01:27Z and 01:01:50Z); those match the triage's recommended labels, with the priority label establishing it as long-term rather than next-release work.
- petr-muller added: "I know @hoxhaeris has looked into ruleset support in Prow" (2026-11-10T01:01:27Z), making hoxhaeris the person with prior art to consult before a design note.
- No PR, issue or remote branch from hoxhaeris mentions rulesets (searched kubernetes-sigs/prow PRs/issues by author and fetched branches), so that investigation is not visible in the repo.

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

### [related-issue] kubernetes-sigs/prow#624 — earlier narrower ruleset issue, superseded by #846
- ref: kubernetes-sigs/prow#624
- relevance: Filed 2026-02-17, #624 asks specifically for the ruleset analogue of `tide.context_options.from-branch-protection` ("`tide` does not respect required contexts from Github Rulesets") and is therefore the clearest statement of the read-only slice. It went `lifecycle/rotten` and was closed `NOT_PLANNED` by `k8s-triage-robot` on 2026-09-09; petr-muller then commented "Discussed in #846" on 2026-10-10, making #846 canonical for that ask rather than #624 being a separate open request.

## Checked
- Repo-wide case-insensitive grep for "ruleset": 6 files, all comment/string-only, no functional code (`pkg/config/tide.go`, `pkg/tide/github.go`, `pkg/tide/status.go`, `pkg/tide/tide_test.go`, `pkg/tide/status_test.go`, `pkg/config/prow-config-documented.yaml`).
- Confirmed Prow does not vendor `go-github`; GitHub client is hand-rolled, so no upstream library ruleset support to adopt.
- Confirmed `cmd/branchprotector`/`pkg/config/branch_protection.go` operate purely on Prow's declarative config, not live GitHub branch-protection/ruleset state (aside from the reconciler's own protection push/diff calls).
- Read kubernetes-sigs/prow#477 in full, including the maintainer's 2026-08-13 comment supplying the missing rationale for #846.
- Grepped `pkg/tide` for `RequiredContexts`/`ContextPolicy`/`contextChecker` to confirm no GitHub branch-protection or rulesets API calls exist in that package.
- Re-checked `upstream/main` (364 commits ahead of `main_sha`) on 2026-11-10: `git grep -i ruleset` still yields 13 hits in the same 6 files, all comment/string/test text (`pkg/config/tide.go:58,258`, `pkg/tide/github.go:775`, `pkg/tide/status.go:266`, tests, documented YAML). No `Ruleset` type, no `/repos/{owner}/{repo}/rulesets` or `/repos/{owner}/{repo}/rules/branches/{branch}` calls; `from-branch-protection` remains the only live-source context option.
- Checked both remotes for merged/open ruleset PRs: `gh search prs` finds only #995 (open, classic branch-protection `require_last_push_approval`/bypass apps, unrelated) and already-triaged #579/#537; no ruleset implementation PR exists.
- Searched for hoxhaeris's referenced prior ruleset work (`gh search prs/issues --author hoxhaeris ruleset`, `git log --all --grep=ruleset --author=hoxhaeris`, remote branch names): no hits. Their recent Prow work is `peribolos`-focused (#1008, #977, #961, #710, #555, #497, #583), consistent with the maintainer's view that ruleset management belongs nearer `peribolos` than `branchprotector`.

## Next steps
- Scope the work using #624's framing for the narrow piece: a ruleset analogue of `tide.context_options.from-branch-protection` so Tide can read required contexts from rulesets; treat full ruleset management as the separate, design-first piece.
- Split into (a) a small verification/test task confirming `GitHubMergeBlocksPolicyMap` correctly handles ruleset-only blocks, and (b) a larger design proposal for ruleset-aware config before any new client/schema work begins.
- Labels are settled: `kind/feature`, `area/tide`, `area/branchprotector`, `priority/important-longterm`. No further label action needed.
- Require a short design note before any new Rulesets API client/config work starts, given the maintainer's explicit concern about repeating branchprotector's configuration-surface problems.
- Link #624's closure ("Discussed in #846") when next commenting, so readers do not reopen the narrower duplicate.
- Ask @hoxhaeris for their prior ruleset investigation notes before drafting the design note; the maintainer flagged it but nothing is in the repo, so the findings would otherwise be lost.

## Advice

Superseded in part at 2026-11-10T01:01Z: petr-muller applied `area/tide`, `area/branchprotector` and `priority/important-longterm`, so items 1 and 3 below are done; item 2 (scope comment) is still open, and should additionally ask @hoxhaeris for their prior ruleset findings.

At the 2026-11-10T00:56:50Z refresh: issue still `OPEN`, only `kind/feature` applied, no assignee or milestone (the repo has no milestones), no comments since 2026-08-13, and no PR in flight (`gh search prs "ruleset"` finds only unrelated open #995). Both cross-references (#477, #624) are already folded into the refresh.

### 1. Add area labels

Rationale: the triage's two angles map exactly onto existing labels; without `area/tide` the issue is not discoverable by the tide maintainers, and `area/branchprotector` ties it to the #477 management half.

```bash
gh issue edit 846 --repo kubernetes-sigs/prow --add-label "area/tide,area/branchprotector"
```

### 2. Post the scope-consolidation comment

Rationale: #624 is closed as not-planned by the lifecycle bot rather than by a maintainer, so the issue currently reads as two dead ends (#477 "wrong issue", #624 "not planned"). A comment recording that the read-only ask lives here, plus the design-first condition from petr-muller's 2026-08-13 comment, is what makes the issue actionable by a contributor. Keep it a comment, not a close of #477.

```bash
gh issue comment 846 --repo kubernetes-sigs/prow --body '> *This was generated by AI during triage.*

## Scope: what #846 covers

Consolidating the related requests:

- kubernetes-sigs/prow#624 asked for the ruleset analogue of `tide.context_options.from-branch-protection` ("tide does not respect required contexts from Github Rulesets"). It was closed as not-planned by the lifecycle bot, not by a maintainer; that read-only ask belongs here.
- kubernetes-sigs/prow#477 (open) is the `branchprotector` "excluded branches retain existing protection" bug that motivated the original report.

Per @petr-muller above, this should start from a short design note rather than code. The note should cover:

- where first-class ruleset management lives (peribolos vs. a new component) and its config surface, including inferring required ProwJobs from `always_run`/`optional` with explicit overrides;
- how Tide consumes that config and GitHub mergeability API surface (`rules/branches/{branch}` looks like the candidate for the read-only slice);
- how the design avoids repeating `branchprotector` configuration-surface problems, including #477.

The read-only Tide piece (recognising ruleset-required contexts) looks separable from ruleset management; the design note can propose splitting it out.' 
```

### 3. Mark it available for a design write-up

Rationale: a design note is bounded, needs no maintainer access, and the maintainer explicitly asked for it before code; `help wanted` signals that to contributors. Do not add `good first issue` — the note requires understanding both Tide's context policy and `branchprotector`'s history.

```bash
gh issue edit 846 --repo kubernetes-sigs/prow --add-label "help wanted"
```

No PR or linked fix exists to review, so there is no review action to suggest. Milestone assignment is not applicable: the repo has no open milestones.

## Open questions
- Should ruleset support be a new parallel config surface, or should `pkg/config/branch_protection.go`'s `Policy` type be extended to also be populated from rulesets?
- Does Prow need to *manage* rulesets (write path, like branchprotector does for classic protection), or only *read/respect* them for Tide's merge and context-policy decisions? The issue title suggests read-only is the immediate need.
- Should the existing `GitHubMergeBlocksPolicyMap`/`mergeStateStatus` path be explicitly tested against ruleset-blocked PRs before any new work starts, to establish how much of this issue is already solved?
