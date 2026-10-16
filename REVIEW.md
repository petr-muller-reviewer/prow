---
pr: kubernetes-sigs/prow#1009
title: "Allow sharding Tide context policies in supplemental config"
head_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
base: main
reviewed_at: 2026-10-15T23:33:49Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-14T15:56:03Z
  gated_head_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
  reviewed_head_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
refresh_log:
  - old_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
    new_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
    at: 2026-10-15T17:32:12Z
    summary: "No code changes; captured three non-blocking inline review comments from @Prucek."
---

# Review

## Verdict

Approve with non-blocking suggestions.

All three independent maintainer perspectives found no blockers: Code Quality and Maintainability recommend approval, maintenance burden is low, and deployment risk is low. The implementation preserves existing configurations and hierarchical inheritance while adding same-scope list unions and boolean conflict detection. The suggestions below are optional. Tests were inspected but not run, and the author's reported test, lint, and large-fixture results were not independently verified. Before adopting shards, upgrade every component consuming them; before rolling back, move supplemental policies back into the main configuration.

## What this PR does

- Accepts Tide context policies in supplemental configuration files.
- Combines context lists and rejects conflicting explicit booleans at matching global, org, repo, and branch scopes.
- Classifies context policies by org and repo for supplemental directory validation.
- Documents configuration flags, shard placement, and inheritance behavior.

## Findings

No blocking or should-fix findings. The following suggestions are optional.

### [nit] Exercise reverse shard order

- where: `pkg/config/tide_test.go:2117-2123`
- concern: The current test checks unions and repeated shards in one order. Consider merging the same valid shards in reverse order and comparing results to directly protect the documented order independence.
- excerpt: |
    incoming := options([]string{"b", "a", "b"})
    var merged TideContextPolicyOptions
    if err := merged.mergeFrom(incoming); err != nil {
        t.Fatal(err)
    }
    if err := merged.mergeFrom(options([]string{"z", "a"})); err != nil {

### [nit] Document the policy-field maintenance requirement

- where: `pkg/config/tide.go:464-466`
- concern: This helper controls scope classification and whether an incoming policy is processed. Consider documenting that adding a serialized policy field also requires updating this helper and the merge logic.
- excerpt: |
    func isTideContextPolicySet(policy TideContextPolicy) bool {
        return policy.SkipUnknownContexts != nil || policy.FromBranchProtection != nil || policy.OverwritePendingContexts != nil ||
            len(policy.RequiredContexts) > 0 || len(policy.RequiredIfPresentContexts) > 0 || len(policy.OptionalContexts) > 0 || len(policy.OptionalRegexContexts) > 0

### [nit] Document upgrade and rollback ordering

- where: `site/content/en/docs/components/core/tide/config.md:171-177`
- concern: Older binaries reject context policies in supplemental files. Consider adding a deployment note to upgrade checkconfig and all configuration consumers before adopting shards, and to consolidate supplemental context policies into the main configuration before rolling back.
- excerpt: |
    `tide.context_options` can be split across supplemental Prow config files. Pass
    `--supplemental-prow-config-dir=config/shards` to Tide and `checkconfig`, alongside
    their main `--config-path`. The directory is searched recursively for filenames
    ending in `_prowconfig.yaml`. Set `--supplemental-prow-configs-filename-suffix`
    to use a different suffix, and repeat `--supplemental-prow-config-dir` to load
    multiple directories. These are centrally managed configuration shards; updating
    a PR's `.prow.yaml` does not configure Tide context policies.

## Resolved

The following suggestions were recorded by earlier reviews. They are retained as history and closed as findings during the earlier defect-first review because they do not demonstrate actionable regressions. This disposition does not claim that code changes addressed them.

### [nit] Include the earlier shard in conflict diagnostics
- where: `pkg/config/tide.go:408-409`
- concern: When two supplemental files set different values for a boolean at the same scope, the error names the scope and field, but not the earlier file that supplied the conflicting value. Including both shard paths would make this easier to diagnose in large shard sets.
- excerpt: |
    if field.a != nil && field.b != nil && *field.a != *field.b {
        errs = append(errs, fmt.Errorf("%s: conflicting values for %s", scope, field.name))
    }

### [nit] Factor policy merging into named helpers
- where: `pkg/config/tide.go:396-412`
- concern: In an inline comment on 2026-10-15, @Prucek suggested moving the closure into a separate function and factoring the merge method, which currently handles policy conflict checks and nested org/repo/branch map traversal. Named helpers could make the merge flow easier to follow.
- excerpt: |
    mergePolicy := func(a, b TideContextPolicy, scope string) TideContextPolicy {
        if !isTideContextPolicySet(b) {
            return a
        }
        for _, field := range []struct {
            name string
            a, b *bool

### [nit] Align the shard-scope test name with the validator
- where: `cmd/checkconfig/main_test.go:2223`
- concern: @Prucek suggested naming the test after `validateAdditionalProwConfigIsInOrgRepoDirectoryStructure` or incorporating it into the existing suite. The current name describes the tested shard-scope behavior, so this is a naming consistency suggestion.
- excerpt: |
    func TestValidateTideContextPolicyShardScope(t *testing.T) {

## Checked

- Three independent reviewers assessed code quality, long-term maintainability, and deployment risk. No concern was independently flagged by multiple reviewers.
- Verified that the checked-out code matches PR head `e7d18e7ad8ef2a600ba9aa778cd600e53993d418`, excluding review artifacts. Local HEAD `c4d61699e162229711763df48843dab362218cb0` adds only a review-artifact commit; `head_sha` identifies the actual reviewed PR head.
- No existing configuration fields, defaults, permissions, external dependencies, or component lifecycle behavior changed.
- Users adopting many small shards may measure reload duration: each nonempty shard rebuilds accumulated maps. No performance regression was demonstrated, so this is an operational consideration rather than a finding.
- Inspected the complete PR diff at `e7d18e7ad8ef2a600ba9aa778cd600e53993d418`; the local HEAD additionally contains a review-artifact commit.
- Followed supplemental loading through `ProwConfig.mergeFrom` and `Tide.mergeFrom`, before defaulting and component validation.
- Checked same-scope list unions, explicit false preservation, boolean conflict checks, nested map allocation, and empty merge-method handling.
- Traced global-to-branch inheritance through `ParseTideContextPolicyOptions` and effective context validation through `GetTideContextPolicy`.
- Checked `HasConfigFor`, global scope detection, and supplemental org/repo directory validation.
- Inspected added tests for loading, inheritance, lists, boolean conflicts, regex conflicts, and shard placement. Tests were not run; runtime behavior remains unverified by this review.
- Earlier gate and refresh metadata are retained as historical records; no new merge gate or CI assessment was performed.

## Open questions

None.
