---
pr: kubernetes-sigs/prow#1009
title: "Allow sharding Tide context policies in supplemental config"
head_sha: b4e4cd937db7ca7ccb25531049b21df8c463d8fb
base: main
reviewed_at: 2026-10-16T12:22:06Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-16T12:32:04Z
  gated_head_sha: b4e4cd937db7ca7ccb25531049b21df8c463d8fb
  reviewed_head_sha: b4e4cd937db7ca7ccb25531049b21df8c463d8fb
refresh_log:
  - old_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
    new_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
    at: 2026-10-15T17:32:12Z
    summary: "No code changes; captured three non-blocking inline review comments from @Prucek."
  - old_sha: e7d18e7ad8ef2a600ba9aa778cd600e53993d418
    new_sha: b4e4cd937db7ca7ccb25531049b21df8c463d8fb
    at: 2026-10-16T12:22:06Z
    summary: "Reviewed two targeted commits extracting merge helpers and consolidating tests; recorded author reply; no new blocking findings."
---

# Review

## Gate

**Decision: merge.** The current PR head `b4e4cd937db7ca7ccb25531049b21df8c463d8fb` matches the refreshed review baseline. @Prucek's three inline comments were addressed by extracting named merge helpers and incorporating hierarchy cases into the existing validator suite. No requested changes or substantive unresolved concerns were found. The remaining local suggestions and reduced regression coverage do not establish a merge blocker.

### Findings disposition

- No unresolved gating findings. The local suggestions about reverse shard order, documenting policy-field maintenance, and upgrade/rollback instructions remain optional.
- @Prucek's closure and factoring comments are addressed in `pkg/config/tide.go:420-480`; the test naming/consolidation comment is addressed by the existing validator suite at `cmd/checkconfig/main_test.go:2016` and its context-policy cases at `2225-2269`.
- @jmguzik confirmed the adjustments on 2026-10-16 at 11:05:46 UTC. @Prucek's submitted review was COMMENTED, with no requested changes.

### Gating list

- None.

### Independent merge risk

- No exported Go API, Kubernetes API, existing configuration field, CLI flag, or default is removed or changed. Existing installations need no migration; new supplemental context policies are opt-in.
- Operators adopting supplemental context policies must upgrade all configuration consumers first. Before rolling back to an older binary, move those policies back to the main configuration because the older supplemental loader rejects this field. Blast radius is limited to installations adopting the new shards; the upgrade/rollback documentation suggestion remains nonblocking.
- Same-scope boolean conflicts reject the candidate configuration; existing hierarchical overrides remain intact. The helper refactor preserves aggregate error propagation and commits the replacement policy only after successful merging.
- Test consolidation removed dedicated regression scenarios, as recorded by the refresh. No corresponding behavior regression was found; tests were inspected but not run.
- Assessed compatibility directly from the diff and existing review evidence. No additional repository compatibility skill was applicable.

## Verdict

Approve with non-blocking suggestions.

All three independent maintainer perspectives found no blockers: Code Quality and Maintainability recommend approval, maintenance burden is low, and deployment risk is low. The implementation preserves existing configurations and hierarchical inheritance while adding same-scope list unions and boolean conflict detection. The suggestions below are optional. Tests were inspected but not run, and the author's reported test, lint, and large-fixture results were not independently verified. Before adopting shards, upgrade every component consuming them; before rolling back, move supplemental policies back into the main configuration.

The 2026-10-16 refresh reviewed the helper extraction and test consolidation at `b4e4cd937`. No new correctness issues were found, and the approval with optional suggestions stands. Some dedicated regression coverage was removed; the retained loading test covers required and required-if-present unions and a branch boolean override.

## What this PR does

- Accepts Tide context policies in supplemental configuration files.
- Combines context lists and rejects conflicting explicit booleans at matching global, org, repo, and branch scopes.
- Classifies context policies by org and repo for supplemental directory validation.
- Documents configuration flags, shard placement, and inheritance behavior.

Since previous review:

- Two commits, `94cdaaf8c` and `b4e4cd937`, changed four files (+179/-260): extracted policy/org/repo merge helpers, incorporated shard hierarchy cases into the existing validator suite, and reduced dedicated tests.
- Deleted `pkg/config/tide_context_sharding_test.go`; moved the loading scenario into `pkg/config/tide_test.go` using `loadConfigYaml`. Dedicated map-independence, all-list union/idempotence, alternate-branch, and inherited-regex-conflict scenarios were removed; inherited behavior remains covered in part by existing tests.
- @jmguzik replied on 2026-10-16 at 11:05:46 UTC that the code was adjusted in response to @Prucek's remarks. No new inline comments or submitted reviews were found. The PR remains open.

## Findings

No blocking or should-fix findings. The following suggestions are optional.

### [nit] Exercise reverse shard order

- where: `pkg/config/tide_test.go:2111-2118`
- concern: The retained loading test supplies supplemental shards in one order. Consider loading the same valid shards in reverse order and comparing effective policies to directly protect the documented order independence.
- excerpt: |
    cfg, err := loadConfigYaml(`tide:
      context_options:
        required-contexts: [global, shared]
        required-if-present-contexts: [conditional, shared-conditional]
        skip-unknown-contexts: true`, t, `tide:
      context_options:
        required-contexts: [root, shared]
        required-if-present-contexts: [extra-conditional, shared-conditional]

### [nit] Document the policy-field maintenance requirement

- where: `pkg/config/tide.go:483-485`
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

The following suggestions were recorded by earlier reviews. They are retained as history and closed as findings during the earlier defect-first review because they do not demonstrate actionable regressions. The dispositions below distinguish suggestions addressed by this refresh from historical suggestions closed without code changes.

### [nit] Include the earlier shard in conflict diagnostics
- where: `pkg/config/tide.go:408-409`
- concern: When two supplemental files set different values for a boolean at the same scope, the error names the scope and field, but not the earlier file that supplied the conflicting value. Including both shard paths would make this easier to diagnose in large shard sets.
- excerpt: |
    if field.a != nil && field.b != nil && *field.a != *field.b {
        errs = append(errs, fmt.Errorf("%s: conflicting values for %s", scope, field.name))
    }

### [nit] Factor policy merging into named helpers

Addressed in `94cdaaf8c`: extracted `mergeSupplementalTideContextPolicy` and org/repo `mergeFrom` helpers. The original location and excerpt below refer to the previously reviewed head.
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

Addressed in `b4e4cd937`: incorporated the shard-scope cases into `TestValidateAdditionalProwConfigIsInOrgRepoDirectoryStructure`. The original location and excerpt below refer to the previously reviewed head.
- where: `cmd/checkconfig/main_test.go:2223`
- concern: @Prucek suggested naming the test after `validateAdditionalProwConfigIsInOrgRepoDirectoryStructure` or incorporating it into the existing suite. The current name describes the tested shard-scope behavior, so this is a naming consistency suggestion.
- excerpt: |
    func TestValidateTideContextPolicyShardScope(t *testing.T) {

## Checked

- Initial review: three independent reviewers assessed code quality, long-term maintainability, and deployment risk. None found a blocker; no concern was independently flagged by multiple reviewers.
- Refreshed the complete delta from `e7d18e7ad8ef2a600ba9aa778cd600e53993d418` to `b4e4cd937db7ca7ccb25531049b21df8c463d8fb`. The change is a targeted refactor and test consolidation, introduces no new concepts, and does not require a full re-review.
- Verified that local HEAD `cc595e145` matches the new PR head except review artifacts; inspected new code directly from the fetched PR commit.
- Checked that extracted helpers preserve boolean conflict aggregation, exact scope diagnostics, list union behavior, nil-map handling, map copies, and assignment only after a successful top-level merge.
- Read retained loading and boolean tests, the shared `loadConfigYaml` helper, and consolidated hierarchy cases. The helper loads actual files using the existing supplemental directories and suffix.
- Original test coverage was broader: this refresh removed dedicated coverage for map independence, all four list types/idempotence, alternate branches/repos, and inherited optional-regex conflicts. This is a residual coverage limitation, not a demonstrated behavior defect.
- Previous checks of loader integration, hierarchical resolution, scope classification, compatibility, and operational risk remain applicable; this refresh changes no configuration fields, flags, defaults, or documentation.
- Tests were inspected but not run. The author's reported test, lint, and large-fixture results remain independently unverified.
- Users adopting many small shards may measure reload duration: each nonempty shard rebuilds accumulated maps. No performance regression was demonstrated.
- The 2026-10-16 gate rechecked prior feedback and compatibility at `b4e4cd937`; see Gate for the current merge recommendation.

## Open questions

None.

## Followups

3 accepted; 2 skipped. PR #1009 merged on 2026-10-16 at 14:08:27 UTC as `e54d32e8b6509869648bde6d5fae18eb75792c88`. Each task targets the default branch containing that merge.

The tests followup incorporates the local checks run after the original review and gate: `go test ./pkg/config -run '^TestSupplementalTideContextMerge' -count=1` passed. These checks found no implementation defect; the test file remains local and untracked. Earlier statements about tests not being run describe the review/gate at those times.

Skipped: upgrade/rollback documentation; the policy-field maintenance comment.

### Retain supplemental Tide context merge regression tests

- Category: tests
- Necessity: should
- Mode: deliver
- Where: `pkg/config/tide_context_merge_regression_test.go; pkg/config/tide_test.go`
- Why followup: The merged PR reduced dedicated coverage. Local follow-on tests passed without implementation changes, but remain untracked in this review worktree and are not upstream.

```text
In kubernetes-sigs/prow, following PR #1009 — "Allow sharding Tide context policies in supplemental config" (merged as e54d32e8b6509869648bde6d5fae18eb75792c88), retain focused regression coverage for supplemental context policy merging on the default branch containing that merge.

Use the existing local pkg/config/tide_context_merge_regression_test.go from the review worktree if available. It contains TestSupplementalTideContextMergeConflictPreservesInputs, TestSupplementalTideContextMergePreservesSiblingMaps, and TestSupplementalTideContextMergeOrderAndIdempotence. If it is unavailable in your checkout, recreate equivalent behavior tests beside pkg/config/tide_test.go using the existing loadConfigYaml helper.

Exercise all three boolean fields at global, org, repo, and branch scopes, in both conflict directions. Check that errors name the exact scope and field, and that failed merges leave the receiver and incoming policy unchanged, including unrelated policy settings. Verify that merging overlapping trees preserves existing and incoming sibling orgs, repos, and branches. Mutate result maps at every level, including one-sided subtrees, and verify neither the previous receiver maps nor incoming maps change. Load three valid shards in all six orders, include repeated and empty shards, and assert sorted deduplicated unions for all four context-list fields and effective policies at global, org, repo, and branch scopes.

Acceptance: focused tests pass with go test ./pkg/config -run '^TestSupplementalTideContextMerge' -count=1 (adjust the expression only if test names change). The existing local tests passed this command against PR head b4e4cd937db7ca7ccb25531049b21df8c463d8fb, with no production changes. Run the package suite after integrating the tests.

Scope: deliver tests only. Keep helpers small and reuse existing fixtures; avoid a generalized testing framework. Do not refactor merge implementations or alter behavior merely to make testing easier. Do not publish or open a PR without a separate explicit request.
```

### Report both source files for supplemental policy conflicts

- Category: diagnostics
- Necessity: could
- Mode: deliver
- Where: `pkg/config/config.go:1792; pkg/config/config.go:3397; pkg/config/tide.go:463`
- Why followup: Current errors identify the incoming file, scope, and field but omit the earlier source. This is optional debugging work for large shard sets, with no demonstrated behavior defect.

```text
In kubernetes-sigs/prow, following PR #1009 — "Allow sharding Tide context policies in supplemental config" (merged as e54d32e8b6509869648bde6d5fae18eb75792c88), improve conflicting supplemental Tide context boolean diagnostics on the default branch containing that merge.

Inspect loadConfig and ProwConfig.mergeFrom in pkg/config/config.go, plus mergeSupplementalTideContextPolicy and the scope-specific mergeFrom methods in pkg/config/tide.go. The loader currently wraps a conflict with the incoming filename, while the policy helper reports only scope and field. Retain enough source provenance during loading to report the file supplying the earlier conflicting explicit value as well as the incoming file. Include the main config path when it supplied the earlier value, and support shards from multiple supplemental directories. Attribute values per exact policy scope and boolean field; inherited booleans at different scopes remain valid overrides.

Acceptance: focused loader regression tests demonstrate that a conflicting global, org, repo, or branch boolean reports both actual source paths, the exact scope, and the field. Include main-config versus shard, shard versus shard, and repeated identical values before a later conflicting value. Equal booleans still merge successfully, list unions remain unchanged, and conflict failures preserve the existing context policy. Run the relevant pkg/config tests.

Scope: diagnostics and internal provenance only. Do not add serialized configuration fields, change defaults or merge semantics, track provenance for unrelated configuration types, or refactor the loader broadly. Keep bookkeeping proportional to this diagnostic need. Do not publish or open a PR without a separate explicit request.
```

### Validate global and org-only policies without known repositories

- Category: validation
- Necessity: should
- Mode: deliver
- Where: `cmd/checkconfig/main.go:1219; cmd/checkconfig/main_test.go:1669; pkg/config/tide.go`
- Why followup: The existing validator enumerates repositories from jobs, branch protection, and repo policies. Without such targets, global or org-only policies can escape validation. This predates the PR but becomes more relevant with independently maintained shards.

```text
In kubernetes-sigs/prow, following PR #1009 — "Allow sharding Tide context policies in supplemental config" (merged as e54d32e8b6509869648bde6d5fae18eb75792c88), close a pre-existing context-policy validation gap on the default branch containing that merge.

Inspect validateTideContextPolicy in cmd/checkconfig/main.go and TestValidateTideContextPolicy in cmd/checkconfig/main_test.go, along with TideContextPolicy.Validate and policy inheritance in pkg/config/tide.go. Validation currently builds repo/branch targets from PresubmitsStatic, BranchProtection.Orgs[*].Repos, and Tide.ContextOptions.Orgs[*].Repos. If those are empty, explicitly configured global or org-only policies are never validated.

Validate the global policy and effective global-plus-org policies even when no concrete repo target exists. Preserve the current effective repo/branch checks and inheritance semantics. Reuse existing validation and policy union logic; make only the smallest additional config-package interface needed if helpers are currently private.

Acceptance: regression tests show that malformed optional-context regexes and conflicting required/required-if-present/optional categories are rejected in global-only and org-only configurations with no static jobs, branch-protection repos, or repo context policies. Include a conflict introduced by combining inherited global and org lists, and valid global/org-only controls. Existing repo/branch checks continue to work. Validation needs no GitHub access, repository cloning, or invented repository/branch names. Run focused checkconfig tests and the affected config/checkconfig package suites.

Scope: close the validation-target gap only. Do not enumerate GitHub repositories, change policy inheritance or runtime merge behavior, introduce broad configuration validation redesigns, or claim this gap was introduced by PR #1009. Do not publish or open a PR without a separate explicit request.
```

