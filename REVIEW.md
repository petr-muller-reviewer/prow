---
pr: kubernetes-sigs/prow#808
title: "override: preserve status base SHA when overriding jobs"
head_sha: 402d33a621297b0172011c6e530351542ef84fb5
base: main
reviewed_at: 2026-08-11T11:09:26Z
verdict: needs-discussion
refresh_log:
  - from: e2d6a6b1ce031b2896302af67b5bb05af12b3098
    to: 402d33a621297b0172011c6e530351542ef84fb5
    summary: Rebase onto updated main (dependabot bumps to golang.org/x/net, x/crypto, x/sys, x/term, x/text, gopkg.in/ini.v1, and new .github/dependabot.yaml); no change to PR content (pkg/plugins/override/* diff against old head is empty). Also noted a new /ok-to-test comment from ylink-lfs (2026-07-29T15:14:37Z), no reviews or inline comments.
  - from: 402d33a621297b0172011c6e530351542ef84fb5
    to: 402d33a621297b0172011c6e530351542ef84fb5
    summary: No code change. Author SaaiAravindhRaja commented 2026-08-01T10:27:47Z asking "is there anything i can improve on?" — still awaiting maintainer response to the should-fix/question findings below.
  - from: 402d33a621297b0172011c6e530351542ef84fb5
    to: 402d33a621297b0172011c6e530351542ef84fb5
    summary: Second full diff pass (2026-08-09), no code change since last review. Tide interaction re-verified and escalated from should-fix to blocking - the stale base SHA fails both Tide gates (context description check at tide.go:1055 AND the pool ProwJob index at tide.go:1974/2171). Two new findings added - duplicate/contradictory comments on the mid-loop error path, and TestHandleCachesFallbackBaseSHA not actually exercising cachedRefGetter.
  - from: 402d33a621297b0172011c6e530351542ef84fb5
    to: 402d33a621297b0172011c6e530351542ef84fb5
    summary: No code change. Prucek commented 2026-08-10T14:17:28Z questioning whether this duplicates #778 ("Add /override-sticky command for persistent overrides", merged 2026-07-14). #778 addresses a different mechanism (a sticky flag that persists overrides across new pushes), not base-SHA preservation on override - the two look complementary rather than duplicative, but worth the author/maintainer confirming explicitly since it's an open question on the PR.
---

## What this PR does

- When `/override` recreates a synthetic successful status/ProwJob for a failed presubmit, it now prefers the base SHA already embedded in the failed status's description (`BaseSHA:` marker, `config.BaseSHAFromContextDescription`) over re-fetching the PR's current base branch SHA.
- Adds `baseSHAForStatus(status, fallback)`: returns the embedded SHA if present, else calls `fallback()`.
- Adds `cachedRefGetter(getter)`: memoizes the fallback `RefGetter` (value and error) so multiple statuses needing the fallback in one invocation trigger only one `GetRef` call.
- Removes the previous unconditional `baseSHAGetter()` call (and its early-return-on-error) at the top of `handle`; the getter is now invoked lazily, per-status, only when needed.
- Implements item 2 of issue #789 (follow-up from #778 review).

Since previous review:
- No code change (head still `402d33a62`). This entry records a second, independent full-diff pass.

## Findings

### [blocking] Stale BaseSHA makes Tide ignore the override entirely, in exactly the scenario this PR targets
- where: `pkg/plugins/override/override.go:536`, effect at `pkg/plugins/override/override.go:564`; consumers at `pkg/tide/tide.go:1055` and `pkg/tide/tide.go:1974`/`2171`
- concern: Both Tide gates for a `/override` result are keyed on the *current* pool base SHA. (1) `prowJobsFromContexts` accepts a green context only if `config.IsSkipRetest(desc) || (baseSHAForContext != "" && baseSHAForContext == baseSHA)`. (2) The synthetic ProwJob is only visible to the pool via `cacheIndexKey(sp.org, sp.repo, sp.branch, sp.sha)`, which indexes on `pj.Spec.Refs.BaseSHA`. Writing the old status's base SHA into both means neither gate matches once the base branch has moved - which is precisely the divergence case the PR exists to handle. Concrete: job fails at base B0, base advances to B1, user runs `/override prow-job`. Pre-PR the status/ProwJob were stamped B1 and Tide counted the context as passing so the PR merged; post-PR both are stamped B0, Tide matches neither, retriggers the job, it fails again, and the override never sticks. Note `/override-sticky` is unaffected because `stickyDescription` (override.go:332) embeds `config.SkipRetestSentinel`, which short-circuits the base-SHA comparison - evidence that plain `/override` deliberately relies on the current base SHA. Needs a maintainer decision: either keep stamping the current base SHA for the status while using the original only for the ProwJob refs, or give plain override an equivalent sentinel-style escape hatch.
- excerpt: |
    baseSHA, err := baseSHAForStatus(status, baseSHAGetter)
    ...
    status.Description = config.ContextDescriptionWithBaseSha(descFn(user), baseSHA)
    // pkg/tide/tide.go:1055
    if config.IsSkipRetest(desc) || (baseSHAForContext != "" && baseSHAForContext == baseSHA) {

### [should-fix] GetRef failure mid-loop leaves a partial override and posts two contradictory comments
- where: `pkg/plugins/override/override.go:517-541`
- concern: Previously `baseSHAGetter()` ran once before any mutation, so a failure returned a single "Cannot get base ref of PR" comment with nothing changed. Now the fallback runs inside the per-status loop. With statuses `[A (has BaseSHA marker), B (no marker)]` and `GetRef` failing: A is fully overridden (ProwJob created, status flipped to success, `done.Insert(A)`), then B hits the error return. The user gets the failure comment, and immediately afterwards the `defer` at override.go:523 posts `successMsg` listing A. Half-applied override plus two comments that contradict each other. Mitigation: resolve the fallback once up front when any targeted status lacks a marker, before mutating anything.
- excerpt: |
    baseSHA, err := baseSHAForStatus(status, baseSHAGetter)
    if err != nil {
        resp := "Cannot get base ref of PR"
        log.WithError(err).Warn(resp)
        return oc.CreateComment(org, repo, number, plugins.FormatResponseRaw(e.Body, e.HTMLURL, user, resp))
    }

### [should-fix] TestHandleCachesFallbackBaseSHA does not exercise the code it names
- where: `pkg/plugins/override/override_test.go:1562-1599`
- concern: `shaGetterFactory` (override.go:701) already memoizes successful lookups (`if baseSHA != "" { return baseSHA, nil }`), so the `getRefCalls == 1` assertion passes unchanged if `cachedRefGetter` is deleted. The test cannot detect a regression in the caching it is named after. The only behavior `cachedRefGetter` genuinely adds is caching an *error* result, which needs a direct unit test with a counting, error-returning `config.RefGetter`.
- excerpt: |
    if got := fc.getRefCalls; got != 1 {
        t.Fatalf("GetRef calls = %d, want 1", got)
    }

### [nit] cachedRefGetter duplicates shaGetterFactory's memoization
- where: `pkg/plugins/override/override.go:520`, `pkg/plugins/override/override.go:609-620`, `pkg/plugins/override/override.go:701-712`
- concern: `shaGetterFactory`'s own comment says it is "a closure to retrieve a sha once" - it already memoizes success. `cachedRefGetter` adds a second layer whose only new effect is caching failures, which is immaterial since the first error aborts `handle`. Either consolidate, or add a comment stating the wrapper exists to make `handle` independent of the caching behavior of whatever `RefGetter` it is given.
- excerpt: |
    func cachedRefGetter(getter config.RefGetter) config.RefGetter {
        var baseSHA string
        var err error
        var called bool
        return func() (string, error) {
            if !called {
                baseSHA, err = getter()
                called = true
            }
            return baseSHA, err
        }
    }

### [nit] No comment on why a status's base SHA can diverge from the PR's current base ref
- where: `pkg/plugins/override/override.go:602-607`
- concern: The rationale (a job's original base SHA differs from today's base-branch tip when the branch moved) lives only in the PR description. A one-line comment on `baseSHAForStatus` would stop a future maintainer reading the marker preference as a bug.

### [question] Is the partial-override-on-error behavior intentional?
- concern: Confirm whether returning mid-loop on `GetRef` failure, after earlier contexts were already mutated and with the deferred success comment still firing, is an accepted tradeoff or should be pre-resolved before any mutation.

### [question] Duplicate event-construction helper in tests
- where: `pkg/plugins/override/override_test.go:1612-1621`
- concern: `overrideCommentEvent` duplicates literal `github.GenericCommentEvent` construction already present at lines 1220, 1506 and 1690 of the same file. Non-blocking, but the existing sites could be migrated to the new helper in the same PR.

## Checked
- `baseSHAForStatus` falls through correctly when `status.Description` is empty - branch-protection-injected synthetic statuses (`github.Status{Context: context}` at override.go:499) have a zero-value Description and `BaseSHAFromContextDescription("")` returns `""`.
- `ContextDescriptionWithBaseSha` is fed only `descFn(user)` as the human-readable part, so re-encoding an already-marked description cannot produce a doubled `BaseSHA:` suffix or blow the 140-char limit.
- `cachedRefGetter` memoizes both value and error on first call; no re-entrancy or goroutine concern (`handle` is single-threaded, matching `shaGetterFactory`'s "not threadsafe" note).
- Checkrun override path is untouched - checkruns carry no base SHA and still use head SHA only.
- `go vet` and `go test ./pkg/plugins/override/` pass on the PR head.
- No config/API/CRD/RBAC surface touched; change confined to `pkg/plugins/override/override.go` and its test. Drop-in and revertible.
- No security-relevant surface: base SHA data originates from Prow's own status descriptions, not user-controlled input.
- Net API-call effect is positive: `GetRef` is skipped entirely when every targeted status carries a marker.

## Open questions
- Was the Tide coupling (`tide.go:1055` context check plus the `cacheIndexKey` pool filter at `tide.go:1974`) considered? With a stale base SHA the override satisfies neither, so `/override` stops unblocking merges in the moved-base case. Is a follow-up (sentinel for plain override, or splitting status SHA from ProwJob refs SHA) planned?
- Should a `GetRef` failure still guarantee all-or-nothing across the contexts of one `/override` invocation, as it did pre-PR?
- Should `TestHandleCachesFallbackBaseSHA` be reworked to test `cachedRefGetter` directly (error caching), since the current assertion passes without it?
- Is this PR complementary to or overlapping with #778 (`/override-sticky`)? Prucek raised this on 2026-08-10; worth an explicit reply distinguishing base-SHA preservation from sticky-override persistence.
