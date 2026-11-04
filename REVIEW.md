---
pr: kubernetes-sigs/prow#998
title: "Allowed Including Label Presets In Presets"
head_sha: de9d6191eabd6f4f35292ade58acf011a74b38f4
base: main
reviewed_at: 2026-11-02T13:31:54Z
verdict: request-changes
refresh_log:
  - old_sha: 2d952766c57dedbe59a8b9d7e9bcba5a3cfb42fc
    new_sha: de9d6191eabd6f4f35292ade58acf011a74b38f4
    summary: Removed the unused mergePreset wrapper; the ambiguous selector finding remains.
  - old_sha: de9d6191eabd6f4f35292ade58acf011a74b38f4
    new_sha: de9d6191eabd6f4f35292ade58acf011a74b38f4
    summary: Recorded two issue comments; no code changes or PR reviews since the prior refresh.
  - old_sha: de9d6191eabd6f4f35292ade58acf011a74b38f4
    new_sha: de9d6191eabd6f4f35292ade58acf011a74b38f4
    summary: Recorded Prucek's approval; no code changes, so the ambiguous-selector finding remains.
---
# Review
## Verdict
Request changes to prevent includes from silently selecting the wrong preset when global and repo-local presets share a label:value selector. Code Quality found no critical Go issues; Maintainability and Deployment Risk independently confirmed this selector ambiguity.

`Prucek` approved on Nov 2, 2026, but the PR head is unchanged, so the ambiguous-selector finding is still unaddressed.

## What this PR does
- Adds `includePresets` selectors to presets.
- Applies included preset data recursively after an outer preset matches a job.
- Sorts includes and reports missing references and cycles.
- Deep-copies the new map and adds coverage for the main include cases.

Since previous review:
- Commit `de9d6191e` removed the unused `mergePreset` wrapper in `pkg/config/jobs.go` (7 lines deleted), resolving the corresponding cleanup finding.

Since previous review (`2026-10-10T15:57:29Z`):
- No code changed; the PR head remains `de9d6191eabd6f4f35292ade58acf011a74b38f4`.
- On Oct 16, 2026, `midnightconman` said tests were passing and asked for a review; on Oct 20, they asked `cjwagner` and `Prucek` to review.

Since previous review (`2026-10-21T00:06:28Z`):
- No code changed; the PR head remains `de9d6191eabd6f4f35292ade58acf011a74b38f4`.
- On Nov 2, 2026, `Prucek` approved the PR; the approval-notifier bot confirmed the approval and noted the `lgtm` label is still required.

## Findings

### [should-fix] Reject ambiguous include selectors across global and repo presets
- where: `pkg/config/config.go:3005-3009`
- concern: Presubmit and postsubmit resolution combines global presets with repo-local presets, but duplicate selector validation only covers the global preset merge. This lookup returns the first match, so an include can silently apply a global preset instead of a repo-local preset with the same selector. Reject ambiguous matches or validate uniqueness across the combined preset list.
- excerpt: |
    func findPresetByLabel(presets []Preset, label, value string) (Preset, bool) {
    	for _, p := range presets {
    		if v, ok := p.Labels[label]; ok && v == value {
    			return p, true
    		}
    	}
    	return Preset{}, false
    }

### [nit] Include more context in missing-reference errors
- where: `pkg/config/config.go:2994-2996`
- concern: The error names the missing selector but not the including preset or include chain. Adding that context would make larger preset graphs easier to diagnose.
- excerpt: |
    included, ok := findPresetByLabel(all, l, preset.IncludePresets[l])
    if !ok {
    	return fmt.Errorf("included preset with label %q not found", pair)
    }

## Resolved

### Remove the unused `mergePreset` wrapper
- where: `pkg/config/jobs.go:69-74`
- resolved: Commit `de9d6191e` removed the unused helper.

## Checked
- Include expansion starts only from a job-matching preset and recursively applies included presets without matching their labels against the job.
- Include keys are sorted; cycle errors include the repeated selector chain; the new map is copied by `DeepCopyInto`.
- Existing configs without `includePresets` retain their behavior. Config parsing disallows unknown fields, so config readers must be upgraded before the new field is used.
- Missing references and cycles return errors from job defaulting. The added tests were read; tests and `go vet` were not run.

## Open questions
- Should ambiguity be rejected in `findPresetByLabel`, or should config validation enforce unique selectors across the combined global and repo-local preset list?

## Followups

Recorded 2026-11-03 against PR head `de9d6191eabd6f4f35292ade58acf011a74b38f4` (PR open, approved, not merged — followup work should start from `main` once it lands). Accepted after walking the candidates: the deferred-review ambiguity fix, the missing-reference error context, and a probe of the real global + in-repo presets path. Skipped: documenting `includePresets` in the presets guide, and renaming the stale `TestMergePreset`.

### [deliver] deferred-review: Reject ambiguous include selectors across global and repo presets
- necessity: should
- where: `pkg/config/config.go:3005-3012` (`findPresetByLabel`), callers at `:2114` and `:2132`

```text
In kubernetes-sigs/prow, following PR #998 ("Allowed Including Label Presets In Presets"), make preset include resolution reject ambiguous selectors. Work from the default branch (main) if the PR has merged; the reviewed PR head is de9d6191eabd6f4f35292ade58acf011a74b38f4.

Context: resolvePresets (pkg/config/config.go:2965) applies every preset whose labels match a job; applyPresetWithIncludes (:2979) resolves each includePresets entry with findPresetByLabel (:3005), which returns the first preset containing the requested label:value and ignores later matches. Presubmits and postsubmits resolve over append(c.Presets, additionalPresets...) (:2114, :2132), where additionalPresets are repo-local presets from .prow.yaml (pkg/config/inrepoconfig.go:290). Central duplicate-selector validation (mergeJobConfigs, :2027) never sees that combined list, so a global preset and a repo-local preset can share a label:value pair.

Task: collect all matching presets instead of returning the first one. In applyPresetWithIncludes, return an error when more than one distinct preset matches an include selector, naming the selector and enough preset detail (for example the candidates' labels) to identify them. Keep existing behavior for unique selectors, missing references, and cycles.

Add tests in pkg/config/jobs_test.go near TestMergePreset: (a) a duplicate selector in the slice passed to resolvePresets errors and names the selector, (b) unique and nested includes still pass.

Acceptance criteria: `go test ./pkg/config/...` and `go vet ./pkg/config/...` pass; an ambiguous include produces a clear error instead of silently choosing a preset.

Scope guard: do not change ordinary preset matching, the duplicate-env/volume conflict behavior, or add load-time validation of the combined preset lists; no unrelated refactors.
```

### [deliver] cleanup: Add the include chain to missing-reference errors
- necessity: could
- where: `pkg/config/config.go:2994-2996`

```text
In kubernetes-sigs/prow, following PR #998, improve the missing-include error in pkg/config/config.go:2994-2996. If the PR has merged, work from main; the reviewed head is de9d6191eabd6f4f35292ade58acf011a74b38f4.

applyPresetWithIncludes currently returns fmt.Errorf("included preset with label %q not found", pair) without the path that led to it, so a failure in a nested include does not say which preset or chain requested it.

Task: include the include chain (the chain slice passed to applyPresetWithIncludes, plus the failing pair) in the error message, for example `preset include a:1 -> b:2: included preset with label "b:2" not found`. Keep the cycle error unchanged.

Add or extend a test in pkg/config/jobs_test.go asserting that a nested missing-reference error contains the chain.

Acceptance criteria: `go test ./pkg/config/...` passes; a nested missing include reports the full chain.

Scope guard: error-message text only; no behavior, ordering, or refactoring changes.
```

### [probe] tests: Exercise include resolution through the real config path
- necessity: could
- where: `pkg/config/inrepoconfig.go:289-295`, `pkg/config/config.go:2108-2148`, `pkg/config/jobs_test.go`

```text
In kubernetes-sigs/prow, probe the preset include machinery added by PR #998. The existing tests call resolvePresets with a single preset slice; exercise the real path where global presets from Config and repo-local presets from .prow.yaml are combined by DefaultAndValidateProwYAML (pkg/config/inrepoconfig.go:289) and defaultPresubmits/defaultPostsubmits (pkg/config/config.go:2108, :2126).

Write throwaway tests (scratch files or uncommitted tests under pkg/config/) covering: (a) an include selector duplicated between a global preset and a repo-local preset, (b) two job-matching outer presets that include the same base preset, (c) a job that matches both an outer preset and a preset it includes, (d) multiple include entries where application order matters, (e) a nested missing reference. Run `go test ./pkg/config/...`.

Triage every failure: real implementation bug or wrong expectation in the test. Apply only the minimal fix for a confirmed bug, and report each failing case. If no real bug is found, leave no files changed.

Acceptance criteria: every listed behaviour exercised; each failure classified with evidence; probe tests deleted before finishing; minimal fix only if a real bug is confirmed.

Scope guard: do not submit the probe tests, do not fix style or unrelated code, and report (rather than silently expanding) any additional bug you find.
```
