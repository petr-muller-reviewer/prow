---
pr: kubernetes-sigs/prow#998
title: "Allowed Including Label Presets In Presets"
head_sha: de9d6191eabd6f4f35292ade58acf011a74b38f4
base: main
reviewed_at: 2026-10-10T15:57:29Z
verdict: request-changes
refresh_log:
  - old_sha: 2d952766c57dedbe59a8b9d7e9bcba5a3cfb42fc
    new_sha: de9d6191eabd6f4f35292ade58acf011a74b38f4
    summary: Removed the unused mergePreset wrapper; the ambiguous selector finding remains.
---
# Review
## Verdict
Request changes to prevent includes from silently selecting the wrong preset when global and repo-local presets share a label:value selector. Code Quality found no critical Go issues; Maintainability and Deployment Risk independently confirmed this selector ambiguity.

## What this PR does
- Adds `includePresets` selectors to presets.
- Applies included preset data recursively after an outer preset matches a job.
- Sorts includes and reports missing references and cycles.
- Deep-copies the new map and adds coverage for the main include cases.

Since previous review:
- Commit `de9d6191e` removed the unused `mergePreset` wrapper in `pkg/config/jobs.go` (7 lines deleted), resolving the corresponding cleanup finding.

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
