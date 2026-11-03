---
pr: kubernetes-sigs/prow#1051
title: "allow merging plank.job_queue_capacities from supplemental Prow config"
head_sha: d1ce7c271f0ee91ac552bc680e296964fb5ed480
base: main
reviewed_at: 2026-11-02T18:58:17Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-11-02T19:01:18Z
  gated_head_sha: d1ce7c271f0ee91ac552bc680e296964fb5ed480
  reviewed_head_sha: d1ce7c271f0ee91ac552bc680e296964fb5ed480
---

# Review

## Gate

**Decision: do-not-merge** — the only blocking item is the stale supplemental-config docs; the code itself is ready.

No commits landed since the review (`d1ce7c2` is still both the reviewed and current head), so every finding in the review stands unchanged. The independent risk pass found no unacceptable merge risk: the change is additive and opt-in, the YAML key and map shape are unchanged, and the exported-field type change is assignability-compatible with `map[string]int`. The PR cannot merge as-is only because `site/content/en/docs/config.md` still tells users that supplemental files support five fields and that other Prow settings belong in the main config.

### Findings disposition

- **Not addressed** — [REVIEW.md, blocking] Supplemental-config docs still claim only five Prow fields are mergeable (`site/content/en/docs/config.md:23-26`). The current head is unchanged; this blocks merge.
- **Not addressed** — [REVIEW.md, should-fix] No YAML-level or checkconfig hierarchy coverage (`pkg/config/config_test.go:8727`, `cmd/checkconfig/main_test.go:2042`). The current head is unchanged. Not itself a merge blocker, but it is the kind of contract test that would catch the docs/allow-list drift.
- **Not addressed (non-gating)** — [REVIEW.md, should-fix] Redundant `map[string]int` conversions (`pkg/config/config.go:2347`, `pkg/plank/controller_test.go:107`, `pkg/plank/reconciler_test.go:417`, `pkg/config/config_test.go:2040`).

### Merge risk

- **Exported API (low)** — `Plank.JobQueueCapacities` changes from `map[string]int` to the new named `config.JobQueueCapacities`. Assignment, indexing, range, and `nil` comparison remain source-compatible; only reflection or type assertions on the old unnamed type would break. Blast radius: external importers of `sigs.k8s.io/prow/pkg/config`.
- **Configuration (low, opt-in)** — `plank.job_queue_capacities` is newly accepted in root supplemental shards. Existing configs parse and behave identically and no default changes. Shards adopting the field require the new binary and `checkconfig`, and must drop it before rolling back.
- **Behavioral (low)** — capacities are merged installation-wide and duplicate queue names are a hard error. No existing valid shard config could use the field before this PR, so there is no regression for current deployments.
- No CRD, Kubernetes API, proto, or flag changes; no repo compatibility skill was applicable.

### Gating list

1. Update `site/content/en/docs/config.md:23-26` to list `plank.job_queue_capacities` and document root-global scope, cross-file uniqueness, and `0`/negative semantics.

## Verdict

Implementation is correct; update the supplemental-config docs (and ideally the loader tests) before merge.

Multi-perspective maintainer review: Code Quality APPROVE, Maintainability COMMENT (LOW burden), Deployment Risk LOW. All three found the merge logic correct and consistent with the existing `SlackReporterConfigs` pattern. The blocker is documentation drift: `site/content/en/docs/config.md` still enumerates only the five pre-existing mergeable fields and says other Prow settings belong in the main config, while this PR makes `plank.job_queue_capacities` legal in root shards. Deployment note: operators must upgrade Prow binaries and `checkconfig` before adding the field to shards, and remove it from shards before rolling back; a duplicate-name error leaves hot-reloaded components on the last good config but fails restarts and `checkconfig`.

## What this PR does

- Introduces a named `config.JobQueueCapacities` type in place of the inline `map[string]int` for `Plank.JobQueueCapacities`, so it can carry a `mergeFrom` method.
- Adds `JobQueueCapacities.mergeFrom`, which adds entries from an additional map and returns an aggregated error for any queue name already present.
- Adds `plank.job_queue_capacities` to the `ProwConfig.mergeFrom` allow-list (`emptyReference`) and updates the "only ... may be set via additional config" error text.
- Wires the new merge into `ProwConfig.mergeFrom`: assign the map when the base is nil, otherwise merge and reject duplicates.
- Adapts existing `pkg/plank` tests to the named type and adds unit + fuzz-property coverage for the merge.

## Findings

### [blocking] Supplemental-config docs still claim only five Prow fields are mergeable
- where: `site/content/en/docs/config.md:23-26`
- concern: CONFIRMED (Maintainability + Deployment Risk, independently). This page tells users exactly which fields a supplemental file may contain and says "Other Prow settings belong in the main config." After this PR that is wrong for `plank.job_queue_capacities`. Update the list and state that the field is root-global only, queue names must be unique across the main config and all shards, and `0` blocks while a negative value removes the limit. The analogous Tide-context-options change (`e7d18e7ad`) updated this same paragraph.
- excerpt: |
    Supplemental files support `branch-protection`, `slack_reporter_configs`,
    `tide.merge_method`, `tide.queries`, and `tide.context_options`. They use the
    same YAML structure as the main config, including the top-level `tide:` key.
    Other Prow settings belong in the main config.

### [should-fix] No YAML-level or checkconfig hierarchy coverage for the new field
- where: `pkg/config/config_test.go:8727` and `cmd/checkconfig/main_test.go:2042`
- concern: Flagged by all three reviewers. `TestJobQueueCapacitiesMergeFrom` and the property test cover the merge helper, and the property test exercises the `emptyReference` allow-list. Nothing exercises YAML → `loadConfig`: a `TestProwConfigMerging` case (main config has queue A, a shard adds B, another repeats A) would catch tag/key or error-wrapping mistakes, and a `validateAdditionalProwConfigIsInOrgRepoDirectoryStructure` case would lock in that a capacities-only shard is classified global (root accepted, org/repo rejected). The latter follows the precedent set for `tide.context_options` in `e7d18e7ad`.
- excerpt: |
    func TestProwConfigMerging(t *testing.T) {

### [should-fix] Explicit `map[string]int` conversions are unnecessary
- where: `pkg/config/config.go:2347`, `pkg/plank/controller_test.go:107`, `pkg/plank/reconciler_test.go:417`, `pkg/config/config_test.go:2040`
- concern: Flagged by Code Quality and Maintainability; verified with a compile check. A named map type and its unnamed underlying type are mutually assignable, and `sets.KeySet` type-infers over the named type. The conversions add diff noise and imply the types are incompatible when they are not.
- excerpt: |
    validJobQueueNames := sets.KeySet[string](map[string]int(c.Plank.JobQueueCapacities))

### [nit] Type comment omits the global/uniqueness/hard-error contract
- where: `pkg/config/config.go:735`
- concern: The comment describes the mapping but not the surprising behavior: capacities are global-only, names must be unique across all files, and a repeat is a hard error rather than an override. That rationale currently lives only in the PR description. Putting it on the type/field comment also carries it into godoc and the generated `prow-config-documented.yaml`.
- excerpt: |
    // JobQueueCapacities maps a job queue name to its maximum concurrency.
    type JobQueueCapacities map[string]int

### [nit] `mergeFrom` is not total, and the nil-base assignment aliases the shard map
- where: `pkg/config/config.go:738-750` and `pkg/config/config.go:3482`
- concern: `jqc[name] = capacity` panics on a nil receiver with a non-nil argument; the only current caller guards that case, and the method mirrors `SlackReporterConfigs.mergeFrom`, so this is robustness rather than a bug. Likewise, assigning `additional.Plank.JobQueueCapacities` directly is harmless because `loadConfig` discards each shard config, but `maps.Clone` would make the method safe if a caller ever retains the additional config.
- excerpt: |
    if pc.Plank.JobQueueCapacities == nil {
        pc.Plank.JobQueueCapacities = additional.Plank.JobQueueCapacities
    } else if err := pc.Plank.JobQueueCapacities.mergeFrom(&additional.Plank.JobQueueCapacities); err != nil {

### [nit] Duplicate-queue error could be more diagnostic
- where: `pkg/config/config.go:744`
- concern: The error names only the later file and the queue, not that the name already exists in the main config or an earlier shard. `loadConfig` does wrap the offending path, so this is adequate today; wording such as "already defined in the main config or another supplemental config" would shorten incident triage.
- excerpt: |
    errs = append(errs, fmt.Errorf("job queue %s passed more than once", name))

## Checked

- Merge semantics: union, duplicate rejection, and a `nil` additional map all behave as described; the only caller guards the nil receiver.
- The `emptyReference` change still rejects every other `Plank` field; the existing "Config not supported by merge logic errors" case covers this, and the error text now includes `plank.job_queue_capacities`.
- The read path in `pkg/plank/reconciler.go:953` is unaffected by the named-type change.
- Configuration compatibility: the YAML key, map shape, and `json:",omitempty"` tag are unchanged, and the named type is assignment-compatible with `map[string]int`; existing configs parse and behave identically.
- `ProwConfig.HasConfigFor`/`hasGlobalConfig`: no code change is needed. The structural diff against `emptyReference` classifies any `Plank` content as global, so `checkconfig` allows the field at the shard root and rejects it under `<org>/` and `<org>/<repo>/`. An explicit test would prevent regression.
- On a load error the config agent logs and keeps the previous config (`pkg/config/agent.go:409`), so hot reload is not taken down by a duplicate-name conflict; restarts and `checkconfig` still fail until fixed.
- `go vet` and the full `pkg/config` suite are green (Code Quality reviewer), as are the focused `pkg/plank` tests locally; CI `pull-prow-unit-test`, `-race-detector-nonblocking`, and `pull-prow-verify-lint` pass. `gofmt`/`git diff --check` are clean.
- `pull-prow-integration` failure is unrelated: `TestRerun` in `test/integration/test/deck_test.go:522` fails waiting for an updated rerun job and then hits a cleanup 404 for ProwJobs. It does not touch ProwConfig merging and looks like the existing timing-sensitive flake.

## Open questions

- Should the duplicate-queue error include the capacity value or the "already defined in the main config or another supplemental config" wording, so operators do not have to hunt through every file?
- Is the nil-receiver/aliasing hardening worth doing, or is mirroring `SlackReporterConfigs.mergeFrom` the intended convention?
- Should `TestHasConfigFor` get an explicit case for a Plank-only shard, given the global classification currently relies on the generic structural fallback?
