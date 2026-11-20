---
issue: kubernetes-sigs/prow#1082
title: "Reusable presets for Tide merge criteria"
state: open
labels: kind/feature, area/tide
main_sha: 9c8eff7c98cc979e833369aefe5d1e86afcf063f
triaged_at: 2026-11-20T00:46:29Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 3
recommended_labels: [kind/feature, area/tide]
---

# Triage

## Verdict

**accepted** — valid, in-scope Prow feature request; already implemented by open PR #1083.

The issue asks for named, reusable Tide query fragments so a centrally managed
baseline of merge criteria does not have to be duplicated across sharded
configs. It is written by a Prow maintainer, includes an example, and states the
design constraints clearly; no duplicate exists. PR #1083 already implements the
recommended design (authored/runtime type split, presets resolved after shard
merge and before dedup, authored YAML preserved on save). Effort is Level 3
(Large) because the change adds a config surface on the shared load/validate
path and changes the Go-visible `TideGitHubConfig.Queries` type.

## What the issue reports

- Tide config can be sharded per org and per repo, which lets repos tweak merge
  criteria but makes it hard to centrally enforce common elements such as
  merge-prohibiting labels and branch treatment.
- Proposed feature: define named `tide.query_presets` (a reusable query
  fragment) and reference them by name from `tide.queries`; a query may use
  multiple presets.
- List fields (`labels`, `missingLabels`, `includedBranches`,
  `excludedBranches`) would be unioned; scalar fields must not conflict, with
  checkconfig enforcing it.
- Presets would be resolved during configuration loading, before the existing
  query deduplication; the authored configuration must survive a load/save
  round-trip without presets resolved.
- The config surface needs preset awareness that runtime `TideQuery` does not,
  so the shared data structure should be split; the issue accepts an internal
  and external Go API break because Prow is not a library.

## Findings

### [cause] Shared merge criteria have no reuse mechanism

- detail: `TideQuery` fuses org/repo selectors with merge criteria, and shard configs are merged by appending queries. There is no named query-fragment concept, so a common baseline must be repeated in every query and shard.
- evidence: `pkg/config/tide.go:660-676`, `pkg/config/tide.go:372-390`

### [related-code] Tide query data model

- where: `pkg/config/tide.go:42`, `pkg/config/tide.go:660-676`
- excerpt: |
    type TideQueries []TideQuery

    type TideQuery struct {
        Author                 string   `json:"author,omitempty"`
        Labels                 []string `json:"labels,omitempty"`
        MissingLabels          []string `json:"missingLabels,omitempty"`
        ExcludedBranches       []string `json:"excludedBranches,omitempty"`
        IncludedBranches       []string `json:"includedBranches,omitempty"`
        Milestone              string   `json:"milestone,omitempty"`
        ReviewApprovedRequired bool     `json:"reviewApprovedRequired,omitempty"`
        Orgs                   []string `json:"orgs,omitempty"`
        Repos                  []string `json:"repos,omitempty"`
        ExcludedRepos          []string `json:"excludedRepos,omitempty"`
    }

### [related-code] Config load, shard merge, validation and dedup path

- where: `pkg/config/config.go:1849-1911`, `pkg/config/config.go:3497-3530`, `pkg/config/config.go:2842-2846`, `pkg/config/config.go:3719-3747`, `pkg/config/tide.go:855-887`
- excerpt: |
    // loadConfig: read main config, then walk additional-prow-config-dir and mergeFrom each shard.
    // After merging: parseProwConfig validates queries, loadConfig computes AllRepos and
    // deduplicateTideQueries keyed by tideQueryConfig.
    // Runtime consumers use TideQueries.QueryMap() / QueryMap.ForRepo.

### [related-code] Existing job-preset mechanism is only adjacent prior art

- where: `pkg/config/jobs.go:45-57`, `pkg/config/config.go:3025-3070`
- excerpt: |
    // Presets can be used to re-use settings across multiple jobs.
    // IncludePresets pulls the env, volumes, volumeMounts, tolerations and affinity
    // of other presets into this one.
- detail: Job presets are label-matched and merge PodSpecs; they do not provide the name-referenced query-fragment semantics requested here.

### [related-pr] PR #1083 implements the requested feature

- ref: kubernetes-sigs/prow#1083
- relevance: Explicitly says "Implements #1082". Adds `TideQuerySpec` / `TideQueryFragment` / `TideQueryPreset`, resolves presets after the shard merge and before dedup, keeps `resolvedQueries` unexported so saved YAML preserves authored queries and preset references, documents the feature, and adds tests. Open, self-approved, currently `/hold` pending further testing and scrutiny.

## Resolved

None — no finding has been answered or invalidated since triage.

## Checked

- Cross-references to #1082 via the issue timeline: only PR #1083.
- Keyword searches over all issue states (`tide preset`, `tide query preset`, `tide query_presets`, `tide config reuse`, `tide queries shard`): no other matching issue or PR.
- Repository scope: Tide code and configuration live in this repo (`pkg/config/tide.go`, `pkg/tide/`, `cmd/tide`, `cmd/checkconfig`).
- `HasConfigFor` (`pkg/config/config.go:3596-3631`) still works with `TideQuerySpec` because selectors stay inline; preset definitions alone define no orgs/repos and therefore do not enable Tide, matching the PR's stated intent.
- Deduplication ordering: PR #1083 resolves presets in `parseProwConfig` before `loadConfig` calls `deduplicateTideQueries`, satisfying the issue's requirement.
- Serialization: resolved queries live in an unexported `resolvedQueries` field exposed via `ResolvedQueries()`, so marshaling emits only authored queries and presets; a round-trip test covers this.
- Test coverage on `main`: query validation and dedup are covered (`pkg/config/tide_test.go`, `pkg/config/config_test.go`, `cmd/checkconfig/main_test.go`); preset behavior is absent. PR #1083 adds `pkg/config/tide_query_spec_test.go` covering round-trip, slice isolation, composition, scalar-presence conflicts, invalid presets, cross-file references, and reload.

## Next steps

- Review and land PR #1083; close #1082 as completed once it merges.
- Resolve the scope inconsistency: the issue says presets are initially global-only and the docs say to define them in the main Prow configuration, but `mergeFrom` accepts `query_presets` from supplemental shards and `TestQueryPresetsAcrossFiles` exercises cross-file definitions. Either restrict definitions to the main config or document that any loaded file may define them (names remain globally unique).
- No label changes needed: `kind/feature` and `area/tide` are already applied and correct. Effort Level 3 means neither `good-first-issue` nor `help-wanted` is appropriate.

## Open questions

- Should `query_presets` be definable only in the main Prow config, or in any loaded supplemental file? PR #1083 currently allows both, while the docs say to define them in the main configuration.
- Do we want naming constraints (length or character set) on preset names beyond non-empty and globally unique?
