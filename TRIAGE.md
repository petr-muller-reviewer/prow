---
issue: kubernetes-sigs/prow#624
title: "`tide` does not respect required contexts from Github Rulesets"
state: closed
labels: kind/feature, lifecycle/rotten, area/tide
main_sha: c4ea9da5a9fbb2c663d4ad449fa6ccb7b037c214
triaged_at: 2026-11-03T22:09:58Z
verdict: wontfix
refresh_log:
  - at: 2026-11-03T22:09:58Z
    since: 2026-09-14T22:28:28Z
    summary: "Incorporated petr-muller's 2026-10-10 comment pointing to #846; re-anchored code checks to upstream main c4ea9da5, where Rulesets support is still absent."
advice:
  advised_at: 2026-11-03T22:10:40Z
  based_on_triaged_at: 2026-11-03T22:09:58Z
---

## Verdict
- bottom-line: `wontfix`; the feature gap remains, but the issue was automatically closed as Not Planned on 2026-09-09 after no maintainer or author follow-up.
- rationale: Tide has branch-protection inference but no Rulesets API client or context-policy option. No linked implementation or decision reopens that work.

## What the issue reports
- Tide can infer required contexts with `tide.context_options.from-branch-protection`.
- GitHub Rulesets can also make contexts required, but Tide does not consume them.
- The reporter asks for Rulesets support because GitHub promotes Rulesets.

Since previous triage:
- 2026-10-10T16:58:23Z: `petr-muller` commented "Discussed in #846", pointing at the open Tide Rulesets design issue; the issue was not reopened.
- No new cross-references or PRs; state remains `CLOSED` / `NOT_PLANNED` with unchanged labels.
- Upstream `main` advanced 181 commits to `c4ea9da5a9fbb2c663d4ad449fa6ccb7b037c214`; re-checked Tide context policy and the GitHub client, still no Rulesets API support.

## Findings

### [reproducibility] Rulesets are not used for Tide context inference
- detail: Current `TideContextPolicy` has `FromBranchProtection` but no Rulesets equivalent. `GetTideContextPolicy` only augments required contexts from `GetBranchProtection`.
- evidence: `pkg/config/tide.go:180-195`, `pkg/config/tide.go:1055-1085` at `c4ea9da5a9fbb2c663d4ad449fa6ccb7b037c214`.

### [cause] Repository client has no Rulesets retrieval method
- detail: `RepositoryClient` declares branch-protection operations, and the concrete client requests `/repos/%s/%s/branches/%s/protection`; no branch Rulesets endpoint is represented in the interface.
- evidence: `pkg/github/client.go:187-196`, `pkg/github/client.go:3000-3006` at `c4ea9da5a9fbb2c663d4ad449fa6ccb7b037c214`; no `Ruleset`-named identifier exists anywhere under `pkg/github/`.

### [related-code] Existing branch-protection path
- where: `pkg/config/tide.go:1074-1084`
- excerpt: |
    if options.FromBranchProtection != nil && *options.FromBranchProtection {
        bp, err := c.GetBranchProtection(org, repo, branch, presubmits)
        ...
        required.Insert(bp.RequiredStatusChecks.Contexts...)
    }
- relevance: A future implementation would need a separately configurable Rulesets path and an explicit coexistence policy with this path.

### [related-code] GitHub reports Ruleset blocking but does not infer checks
- where: `pkg/tide/github.go:775`, `pkg/tide/status.go:266`
- detail: Tide's merge/status messages mention Rulesets as a possible GitHub-side block; that diagnostic handling does not populate Tide's required-context set.

### [related-issue] Maintainer points to #846 as the active design thread
- ref: kubernetes-sigs/prow#846
- relevance: `petr-muller` commented "Discussed in #846" on 2026-10-10T16:58:23Z. #846 remains open, with no comments after 2026-08-13T12:36:00Z and no recorded design conclusion or implementation.

### [related-issue] Automatic closure as Not Planned
- ref: kubernetes-sigs/prow#624
- relevance: `k8s-triage-robot` issued `/close not-planned` at 2026-09-09T14:23:18Z, and `kubernetes-prow` closed the issue at 2026-09-09T14:23:27Z. No human comment or linked PR followed the original report.

## Checked
- Current issue state (`CLOSED` / `NOT_PLANNED`), labels, comments, and timeline through 2026-11-03T22:09:58Z; no new cross-reference events or linked PRs.
- Upstream `main` at `c4ea9da5a9fbb2c663d4ad449fa6ccb7b037c214` (181 commits past the prior baseline) for Tide context-policy fields and GitHub client API coverage.
- Ruleset-related strings in Tide are diagnostic text, not Rulesets API support.

## Next steps
- Leave closed unless a contributor intends to implement and maintain Rulesets support.
- The #846 cross-reference is now recorded on #624; the earlier suggestion to comment on #846 adds little unless a concrete design proposal appears there.
- If revived, reopen the issue and propose an opt-in context-policy field, Rulesets API types/client method, tests, and a documented union behavior with branch protection.

## Advice
- No action needed. #624 is already `CLOSED` / `NOT_PLANNED`, `petr-muller` recorded "Discussed in #846" on 2026-10-10T16:58:23Z, and there have been no comments, label changes, or new cross-references since triage.
- Do not post another #846 comment: #846 has had no comments since 2026-08-13T12:36:00Z, so the earlier suggested comment has no new design discussion to attach to.
- If #846 produces a design proposal for Rulesets-backed required-context inference, reopen #624 or cite its use case in that proposal.
- Background: open `kubernetes-sigs/prow#994` covers branchprotector versus Rulesets planning; closed `kubernetes-sigs/prow#575` was a reverted `Restrict updates` attempt. Neither implements required-context inference.

## Open questions
- Is there a maintainer or contributor prepared to own a Rulesets API integration and its authentication/compatibility behavior?
- If reopened, should Rulesets-derived contexts be unioned with branch-protection contexts when both are enabled?
