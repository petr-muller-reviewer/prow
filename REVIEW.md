---
pr: kubernetes-sigs/prow#746
title: "tide: fix status reporting for missing newly required contexts"
head_sha: 1b5bbbf89c719791022f7ae35b6926db186764b3
base: main
reviewed_at: 2026-06-12T15:45:34Z
verdict: request-changes
state: merged
refresh_log:
  - from: 6770dc470923e3dc1f6a2d3999fa37d411a69daf
    to: 1b5bbbf89c719791022f7ae35b6926db186764b3
    at: 2026-06-12T15:45:34Z
    summary: PR merged after matthyx APPROVED (2026-06-12); no tide code changes between SHAs; blocking double-counting finding was not addressed before merge
---

## Since previous review

- 2026-06-10 to 2026-06-12: author posted three `/retest` retriggers (CI); no code changes.
- 2026-06-12T10:18: author pinged @matthyx and @cjwagner for review.
- 2026-06-12T10:28: matthyx **APPROVED** ("looks good, thanks") without flagging the blocking double-counting finding. The `[APPROVALNOTIFIER]` bot marked the PR approved.
- PR **merged** into `main` at `1b5bbbf89c719791022f7ae35b6926db186764b3`. No substantive code changes to `pkg/tide/` between the reviewed SHA and merge.

## Findings

### [blocking] Double-counting of missing required contexts in requirementDiff
- where: `pkg/tide/status.go:235-242`
- concern: `unsuccessfulContexts` (tide.go:878) already calls `cc.MissingRequiredContexts` and returns missing contexts as failed entries. The new explicit call at status.go:242 adds them again. Every missing required context is counted twice in `diff`. The `slices.Compact` at line 248 fixes the description string but runs after `diff += len(contexts)` at line 245, so the diff value remains inflated. `diff` is used at status.go:322 to rank PR-to-query matches (`minDiffCount`), so inflation affects which query Tide considers closest for a PR. All three independent reviewers (code quality, maintainability, deployment risk) converged on this issue.
- excerpt: |
    for _, ctx := range unsuccessfulContexts(headContexts, cc, log) {
        contexts = append(contexts, string(ctx.Context))
    }
    presentContexts := sets.New[string]()
    for _, ctx := range headContexts {
        presentContexts.Insert(string(ctx.Context))
    }
    contexts = append(contexts, cc.MissingRequiredContexts(presentContexts.UnsortedList())...)

### [blocking] Test expectedDiff values encode the double-counting bug
- where: `pkg/tide/status_test.go:1324-1435`
- concern: Test comments misattribute the expectedDiff values. E.g., "Missing required context not present in PR" claims `expectedDiff: 2` is "1 for ci/test being present + 1 for missing ci/required-check" but ci/test is successful and not counted; the 2 comes from ci/required-check being counted twice. Tests pass but validate incorrect behavior. Flagged independently by code quality and maintainability reviewers.
- excerpt: |
    expectedDiff:         2, // 1 for ci/test being present + 1 for missing ci/required-check

### [blocking] Duplicated responsibility creates maintenance hazard
- where: `pkg/tide/status.go:235-242` and `pkg/tide/tide.go:878`
- concern: `MissingRequiredContexts` is now called in two places for the same data flow — inside `unsuccessfulContexts` and again explicitly in `requirementDiff`. Future maintainers modifying context-checking logic must trace both call sites and understand their interaction, with `Compact` acting as a silent safety net. The `contextCheckerGetterFactory` append fix alone may be the complete root cause fix: once the factory correctly populates `RequiredContexts`, `unsuccessfulContexts` naturally detects missing ones without needing the second call. If both calls are genuinely needed, a comment explaining why is required.

### [should-fix] Unnecessary len guard on append
- where: `pkg/tide/status.go:793-795`
- concern: `append(slice, empty...)` is a no-op in Go. The `if len(requiredContexts) > 0` guard adds no value and obscures the intent.
- excerpt: |
    if len(requiredContexts) > 0 {
        contextPolicy.RequiredContexts = append(contextPolicy.RequiredContexts, requiredContexts...)
    }

### [should-fix] TestContextCheckerGetterFactoryAppend uses type assertion on internal type
- where: `pkg/tide/status_test.go:1612-1616`
- concern: Test type-asserts to `*config.TideContextPolicy` and inspects `RequiredContexts` slice directly. If the return type ever becomes a wrapper, this breaks for reasons unrelated to the tested behavior. Testing through the `contextChecker` interface (calling `MissingRequiredContexts` and `IsOptional`) would be more robust.
- excerpt: |
    policy, ok := cc.(*config.TideContextPolicy)
    if !ok {
        t.Fatalf("Expected *config.TideContextPolicy, got %T", cc)
    }

### [nit] Whitespace-only changes in unrelated code
- where: `pkg/tide/status.go:48-50`, `pkg/tide/status_test.go:1751,1764`
- concern: Alignment fixes for constant declarations and map literals unrelated to this PR. Creates diff noise.

### [nit] Nil git.ClientFactory in TestContextCheckerGetterFactoryAppend
- where: `pkg/tide/status_test.go:1601`
- concern: `var gc git.ClientFactory` is nil. Works because the tested code path doesn't call git methods, but will panic if factory internals change.
- excerpt: |
    var gc git.ClientFactory

## Checked
- `contextCheckerGetterFactory` append-vs-replace fix is correct and addresses the original issue of base required contexts being lost
- `slices` and `sets` imports are already present and used elsewhere in the file
- `MissingRequiredContexts` method correctly computes set difference of RequiredContexts minus present contexts
- `slices.Compact` correctly deduplicates sorted slices (called after `sort.Strings`)
- No security concerns
- Deployment risk is low: no config schema changes, no new API calls, backward-compatible, safe to roll back
- Previously silently dropped contexts from Tide config (overwritten by ProwJob-derived contexts) will now be correctly enforced — this is the desired behavior

## Open questions
- Was the double-counting in `requirementDiff` intentional as a stronger penalty for missing required contexts, or an oversight from not noticing that `unsuccessfulContexts` already handles this?
- Would removing the new `MissingRequiredContexts` call from `requirementDiff` entirely (relying on `unsuccessfulContexts`) be acceptable, since that function already covers the missing-context case?
- If both calls are genuinely needed (e.g., `requirementDiff` is sometimes called with a `contextChecker` that does not go through `contextCheckerGetterFactory`), can that reason be documented with a comment?
