---
issue: kubernetes-sigs/prow#736
title: "Tide status controller falsely reports \"In merge pool.\" for PRs blocked by a never-triggered required context"
state: open
labels: ""
main_sha: 9fa65c208d31319fc1d64b01100656a8c7a52199
triaged_at: 2026-06-08T19:00:57Z
verdict: accepted
---

## Findings

### [cause] contextCheckerGetterFactory overwrites RequiredContexts with nil for filtered-out PRs
- detail: `contextCheckerGetterFactory` (`status.go:787`) unconditionally assigns `contextPolicy.RequiredContexts = requiredContexts`, where `requiredContexts` comes from `requiredContextsMap(filteredPools)`. Filtered-out PRs have no entry in this map, so the config-derived policy is replaced with nil. `MissingRequiredContexts` (`config/tide.go:1026`) then early-returns nil.
- evidence: `pkg/tide/status.go:787`, `pkg/tide/tide.go:814-826`, `pkg/config/tide.go:1025-1038`

### [cause] requirementDiff only sees contexts already posted to the commit
- detail: `requirementDiff` calls `unsuccessfulContexts` which iterates `commit.Commit.Status.Contexts` — GitHub statuses posted to the commit. A required job that never ran has no status, so it is invisible. `unsuccessfulContexts` does call `MissingRequiredContexts` at `tide.go:878`, but the first cause zeroed out the list.
- evidence: `pkg/tide/status.go:228-248`, `pkg/tide/tide.go:865-889`

### [cause] Combined effect produces false "In merge pool."
- detail: With RequiredContexts zeroed (cause 1) and the absent context invisible on the commit (cause 2), `requirementDiff` returns `diffCount == 0`, `hasFulfilledQuery = true` at `status.go:319`. The code falls through to `MissingRequiredContexts(passingUpToDateContexts)` at line 357 — still zeroed. Result: `StatusSuccess` with "In merge pool."
- evidence: `pkg/tide/status.go:270-361`

### [reproducibility] Triggered by adding a required presubmit to a repo with open PRs
- detail: When a presubmit is changed from optional/`always_run: false` to `always_run: true`, existing open PRs with all other requirements met will never have triggered the new job. The sync loop correctly filters them out, but the status controller reports them as in the merge pool.
- evidence: https://github.com/openshift/operator-framework-operator-controller/pull/737 after https://github.com/openshift/release/pull/79764

### [related-code] contextCheckerGetterFactory — Gap 2 overwrite
- where: `pkg/tide/status.go:781-789`
- excerpt: |
    contextPolicy, err := cfg.GetTideContextPolicy(gc, org, repo, branch, baseSHAGetter, headSHA)
    if err != nil {
        return nil, err
    }
    contextPolicy.RequiredContexts = requiredContexts
    return contextPolicy, nil

### [related-code] requirementDiff — Gap 1 context enumeration
- where: `pkg/tide/status.go:228-248`
- excerpt: |
    for _, commit := range pr.Commits.Nodes {
        if commit.Commit.OID == pr.HeadRefOID {
            for _, ctx := range unsuccessfulContexts(append(commit.Commit.Status.Contexts, checkRunNodesToContexts(log, commit.Commit.StatusCheckRollup.Contexts.Nodes)...), cc, log) {
                contexts = append(contexts, string(ctx.Context))
            }
        }
    }

### [related-code] setStatuses passes nil requiredContexts for filtered PRs
- where: `pkg/tide/status.go:429-459`
- excerpt: |
    cr := contextCheckerGetterFactory(c, sc.gc, org, repo, branch, baseSHAGetter, headSHA, requiredContexts[prKey(pr)])

### [related-code] requiredContextsMap only iterates filtered-in PRs
- where: `pkg/tide/tide.go:814-826`
- excerpt: |
    func requiredContextsMap(subpoolMap map[string]*subpool) map[string][]string {
        requiredContextsMap := map[string][]string{}
        for _, sp := range subpoolMap {
            for _, pr := range sp.prs {
                requiredContextsSet := sets.Set[string]{}
                for _, requiredJob := range sp.presubmits[pr.Number] {
                    requiredContextsSet.Insert(requiredJob.Context)
                }
                requiredContextsMap[prKey(&pr)] = sets.List(requiredContextsSet)
            }
        }
        return requiredContextsMap
    }

### [related-code] MissingRequiredContexts early-returns nil
- where: `pkg/config/tide.go:1025-1038`
- excerpt: |
    func (cp *TideContextPolicy) MissingRequiredContexts(contexts []string) []string {
        if len(cp.RequiredContexts) == 0 {
            return nil
        }

### [related-code] GetTideContextPolicy correctly computes RequiredContexts
- where: `pkg/config/tide.go:944-995`
- excerpt: |
    // This correctly derives required contexts from presubmit config,
    // but the result is overwritten by contextCheckerGetterFactory.

### [related-code] unsuccessfulContexts calls MissingRequiredContexts
- where: `pkg/tide/tide.go:865-889`
- excerpt: |
    for _, c := range cc.MissingRequiredContexts(contextsToStrings(contexts)) {
        failed = append(failed, newExpectedContext(c))
    }

### [related-code] sync loop stores requiredContextsMap from filteredPools
- where: `pkg/tide/tide.go:577-584`
- excerpt: |
    filteredPools := c.filterSubpools(c.provider.isAllowedToMerge, rawPools)
    c.statusUpdate.requiredContexts = requiredContextsMap(filteredPools)

### [related-code] Test gap — all requiredContexts tests use inPool=true
- where: `pkg/tide/status_test.go:525-710`
- excerpt: |
    // All test cases with requiredContexts field set inPool: true.
    // No test covers inPool=false with config-derived required contexts.

### [related-code] TestSetStatusRespectsRequiredContexts — inPool=true only
- where: `pkg/tide/status_test.go:2009-2066`
- excerpt: |
    pool := map[string]CodeReviewCommon{prKey(crc): *crc}
    sc.setStatuses([]CodeReviewCommon{*crc}, pool, blockers.Blockers{}, nil, requiredContexts)

## Checked
- Verified all code locations cited in the issue against current HEAD (9fa65c2): line numbers shifted (issue says 762, now 787) but code is identical
- Confirmed `unsuccessfulContexts` calls `MissingRequiredContexts` (tide.go:878) — Gap 1 is effectively fixed once Gap 2 is fixed
- Confirmed no existing test covers inPool=false with requiredContexts from config
- Checked `filterPR` (`pkg/tide/tide.go:755-793`) — correctly filters PRs with unsuccessful contexts using the subpool's own context checker, not the overwritten one
- Confirmed @valen-mascarenhas14 self-assigned the issue on 2026-06-08

## Next steps
- Apply labels: `kind/bug`, `area/tide`, `help-wanted`
- Reply to @valen-mascarenhas14 with contributor guidance: fix Gap 2 with nil-guard in `contextCheckerGetterFactory` (`status.go:787`), add `TestExpectedStatus` case with `inPool: false` + config-derived required contexts
- Decide whether Gap 1 defense-in-depth fix belongs in the same PR or a follow-up

## Open questions
- Should the fix use a nil-guard (`if requiredContexts != nil`) or merge (`append`)? Nil-guard is simpler, preserves override semantics for in-pool PRs. Merge is more defensive but may produce duplicates.
- Is Gap 1 defense-in-depth worth pursuing separately, or is fixing Gap 2 sufficient since `unsuccessfulContexts` already calls `MissingRequiredContexts`?
