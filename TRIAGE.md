---
issue: kubernetes-sigs/prow#624
title: "`tide` does not respect required contexts from Github Rulesets"
state: closed
labels: kind/feature, lifecycle/rotten, area/tide
main_sha: d91e0f84e0f2050b41390b28b69a05f5284ba4a2
triaged_at: 2026-09-14T22:28:28Z
verdict: wontfix
---

## Verdict
- bottom-line: `wontfix`; the feature gap remains, but the issue was automatically closed as Not Planned on 2026-09-09 after no maintainer or author follow-up.
- rationale: Tide has branch-protection inference but no Rulesets API client or context-policy option. No linked implementation or decision reopens that work.

## What the issue reports
- Tide can infer required contexts with `tide.context_options.from-branch-protection`.
- GitHub Rulesets can also make contexts required, but Tide does not consume them.
- The reporter asks for Rulesets support because GitHub promotes Rulesets.

## Findings

### [reproducibility] Rulesets are not used for Tide context inference
- detail: Current `TideContextPolicy` has `FromBranchProtection` but no Rulesets equivalent. `GetTideContextPolicy` only augments required contexts from `GetBranchProtection`.
- evidence: `pkg/config/tide.go:179-194`, `pkg/config/tide.go:941-971` at `d91e0f84e0f2050b41390b28b69a05f5284ba4a2`.

### [cause] Repository client has no Rulesets retrieval method
- detail: `RepositoryClient` declares branch-protection operations, and the concrete client requests `/repos/%s/%s/branches/%s/protection`; no branch Rulesets endpoint is represented in the interface.
- evidence: `pkg/github/client.go:174-188`, `pkg/github/client.go:2839-2864` at `d91e0f84e0f2050b41390b28b69a05f5284ba4a2`.

### [related-code] Existing branch-protection path
- where: `pkg/config/tide.go:960-971`
- excerpt: |
    if options.FromBranchProtection != nil && *options.FromBranchProtection {
        bp, err := c.GetBranchProtection(org, repo, branch, presubmits)
        ...
        required.Insert(bp.RequiredStatusChecks.Contexts...)
    }
- relevance: A future implementation would need a separately configurable Rulesets path and an explicit coexistence policy with this path.

### [related-code] GitHub reports Ruleset blocking but does not infer checks
- where: `pkg/tide/github.go:633`, `pkg/tide/status.go:265`
- detail: Tide's merge/status messages mention Rulesets as a possible GitHub-side block; that diagnostic handling does not populate Tide's required-context set.

### [related-issue] Automatic closure as Not Planned
- ref: kubernetes-sigs/prow#624
- relevance: `k8s-triage-robot` issued `/close not-planned` at 2026-09-09T14:23:18Z, and `kubernetes-prow` closed the issue at 2026-09-09T14:23:27Z. No human comment or linked PR followed the original report.

## Checked
- Current issue state, labels, comments, and issue events through 2026-09-14T22:28:28Z; no linked PR or cross-reference was returned.
- Current upstream `main` at `d91e0f84e0f2050b41390b28b69a05f5284ba4a2` for Tide context-policy fields and GitHub client API coverage.
- Ruleset-related strings in Tide are diagnostic text, not Rulesets API support.

## Next steps
- Leave closed unless a contributor intends to implement and maintain Rulesets support.
- If revived, reopen the issue and propose an opt-in context-policy field, Rulesets API types/client method, tests, and a documented union behavior with branch protection.

## Open questions
- Is there a maintainer or contributor prepared to own a Rulesets API integration and its authentication/compatibility behavior?
- If reopened, should Rulesets-derived contexts be unioned with branch-protection contexts when both are enabled?
