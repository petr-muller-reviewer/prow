---
pr: kubernetes-sigs/prow#1059
title: "config: preserve single-file job configuration settings"
head_sha: 6052db28fe8167c7617ba2c82a4defa18bb948e3
base: main
reviewed_at: 2026-11-07T22:51:39Z
verdict: request-changes
gate:
  decision: hold
  gated_at: 2026-11-07T22:52:59Z
  gated_head_sha: 6052db28fe8167c7617ba2c82a4defa18bb948e3
  reviewed_head_sha: 6052db28fe8167c7617ba2c82a4defa18bb948e3
---

# Review

## Gate

**hold** — the single-file fix works, but it drops duplicate-preset validation for one-file configs and multi-file configs still ignore the same fields.

The PR head is unchanged since the review (`6052db2`), no reviewer feedback has been posted, and neither `REVIEW.md` finding is addressed. The change is a net improvement for `decorate_all_jobs` and `prow_ignored` on single-file loads, but it is not safe to merge as-is because it silently accepts duplicate preset `label:value` pairs that the base commit rejects.

Gating items:

- `pkg/config/config.go:1841-1850` (`REVIEW.md`, should-fix): skipping `mergeJobConfigs` for the first file removes the only duplicate-preset validation; single-file configs that previously failed now load. Fix by merging as before and copying `DecorateAllJobs`/`ProwIgnored` from `subConfig` when `r.count == 0`.
- `pkg/config/config.go:2085-2090` (`REVIEW.md`, question): multi-file configs still discard both fields from every file; confirm intended scope or carry the first file's values forward.

Merge risk:

- Behavioral: single-file deployments that set `decorate_all_jobs: true` or `prow_ignored` will now have those values honored instead of dropped — the intended bugfix, but it changes decoration for every job in such configs. Blast radius is limited to single-file job config deployments; release-note it.
- No exported API, config-schema, or flag changes; the `ReadJobConfigs` signature is unchanged.

## Verdict

Request changes: the single-file case is fixed, but it also silently disables duplicate-preset validation for one-file job configs.

`readFile` now skips `mergeJobConfigs` entirely when the file is the first (or only) one. That restores `DecorateAllJobs` and `ProwIgnored`, but `mergeJobConfigs` is also the only place that rejects two presets sharing a `label:value` pair, so a single-file config with duplicate presets that was rejected before now loads. Keeping the merge and carrying the two fields over from the first file fixes the reported bug without losing the validation.

## What this PR does

- Fixes `ReadJobConfigs` dropping file-level `decorate_all_jobs` and `prow_ignored` when the job config resolves to exactly one file.
- Changes `jobConfigReader.readFile` to assign the first parsed `JobConfig` directly and call `mergeJobConfigs` only for subsequent files.
- Extends `TestReadJobConfigsFileAndDirectory` to assert both fields, adding an empty-directory-plus-file case that reproduces the regression.

## Findings

### [should-fix] Duplicate-preset validation is skipped for single-file configs
- where: `pkg/config/config.go:1841-1850`
- concern: `mergeJobConfigs` contains the only duplicate-preset `label:value` check in the tree. By not calling it for the first file, a config that resolves to a single file no longer rejects presets that collide, while the same file still fails once any second file exists. On the base commit both a bare file and a directory containing one file return `duplicated preset 'label:value' pair : foo:bar`; with this change both return `nil`. Merging as before and then copying `DecorateAllJobs`/`ProwIgnored` from `subConfig` when `r.count == 0` would keep the validation and still fix the bug.
- excerpt: |
    // Merging into an empty config would discard file-level settings.
    jc := subConfig
    if r.count > 0 {
        var err error
        jc, err = mergeJobConfigs(r.jc, subConfig)
        if err != nil {
            r.errs = append(r.errs, err)
            return
        }
    }

### [question] Multi-file configs still silently drop these settings
- where: `pkg/config/config.go:2085-2090`
- concern: `mergeJobConfigs` starts from `JobConfig{}` and copies only presets and the three job lists, so once a second file exists the first file's `decorate_all_jobs` and `prow_ignored` are still discarded. I confirmed with two directories: the one setting `decorate_all_jobs: true` is ignored when another file is also loaded. The commit message says "preserve the first parsed config and merge only subsequent files", but that only holds when exactly one file is loaded. Is multi-file behavior intentionally left as-is (follow-up), or should the first file's values be carried forward and conflicting definitions rejected?
- excerpt: |
    func mergeJobConfigs(a, b JobConfig) (JobConfig, error) {
        // Merge everything.
        // *** Presets ***
        c := JobConfig{}
        c.Presets = append(a.Presets, b.Presets...)

## Checked

- `DecorateAllJobs` and `ProwIgnored` are the only JSON-backed `JobConfig` fields `mergeJobConfigs` drops; `AllRepos` and the `ProwYAMLGetter*` fields are runtime-only.
- The new test case fails on the base commit with `DecorateAllJobs: false` and `ProwIgnored: nil`, so it does capture the reported regression.
- `go test ./pkg/config -count=1` passes on the PR head.
- The new test's empty-directory-plus-file setup exercises the same `r.count == 0` path as a plain single-file or single-file-directory load.

## Open questions

- Should a second file that also sets `decorate_all_jobs` or `prow_ignored` be an error, be ignored, or override the first? Whatever is chosen should be documented on `ReadJobConfigs`.
