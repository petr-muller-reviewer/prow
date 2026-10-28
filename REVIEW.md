---
pr: kubernetes-sigs/prow#237
title: "Add Orgs sharding for Blunderbuss"
head_sha: 5487a39ae18af55a4c13b258e16bc4e38b94f24b
base: main
reviewed_at: 2026-10-20T18:07:50Z
verdict: request-changes
refresh_log:
  - from: 4e881b4c27b4f5e11c167e52800b513b7706abc0
    to: 4e881b4c27b4f5e11c167e52800b513b7706abc0
    summary: >-
      No code change; re-reviewed via multi-perspective maintainer-review
      (code-quality, maintainability, deployment-risk agents + advisor
      synthesis). Confirmed the descendShards blocking bug independently
      across all three perspectives, refuted the earlier WaitForStatus/
      DescriptionRe nil-panic sub-claim, and added two new findings
      (repo-key convention divergence from Bugzilla, genyaml changes bundled
      without explanation).
  - from: 4e881b4c27b4f5e11c167e52800b513b7706abc0
    to: 5487a39ae18af55a4c13b258e16bc4e38b94f24b
    summary: >-
      Updated the review after the force-push/rebase. descendShards now writes
      modified repo shards back to the map, resolving the defaulting bug; a
      separate doc-generator commit explains the omitempty-int documentation
      updates. Other findings remain open.
---

## What this PR does

- Moves `Blunderbuss` plugin config fields (`ReviewerCount`, `MaxReviewerCount`, `ExcludeApprovers`, etc.) into a new embedded `*BlunderbussConfig` struct.
- Adds `Blunderbuss.Orgs map[string]BlunderbussOrgConfig` and `BlunderbussOrgConfig.Repos map[string]BlunderbussConfig` to allow org- and repo-scoped overrides.
- Adds `Configuration.BlunderbussFor(org, repo)` resolving precedence repo > org > global, and wires all four blunderbuss event handlers (`handlePullRequestEvent`, `handleGenericCommentEvent`, `handleStatusEvent`, plus help provider) to use it instead of the flat `pc.PluginConfig.Blunderbuss`.
- Adds `Blunderbuss.SetDefaults()`, `descendShards()`, `mergeFrom()`, and a per-shard `compileRegexpsAndDurations()` to apply defaults/validation/regex-compilation/config-merging across global+org+repo shards, mirroring the existing Bugzilla sharding pattern.
- Extends `genyaml`/`populate_struct` to handle pointer-embedded structs (`ast.StarExpr`) and `omitempty` int fields, needed because `BlunderbussConfig` is now referenced via pointer; regenerates doc YAML (incidentally populating other previously-blank `omitempty` int fields across the docs, unrelated to Blunderbuss).

Since previous review:
- `pkg/plugins/config.go` now writes the modified repo shard back into `orgConfig.Repos`, so defaults applied by `descendShards` persist.
- Commit `fe7826a84` adds `omitempty` integer population to `pkg/genyaml/populate_struct.go` and regenerates the corresponding config docs.

## Findings

### [should-fix] pointer value formatted with %d in validation error messages
- where: `pkg/plugins/config.go:1539` and `pkg/plugins/config.go:1549`
- concern: `b.ReviewerCount` is `*int`, but is passed directly to `%d` in two `fmt.Errorf` calls instead of being dereferenced (or using the already-computed local `reviewerCount`). This produces an unreadable error like `%!d(*int=0xc0001234)` instead of the intended integer. The original code correctly used `*b.ReviewerCount`.
- excerpt: |
    return fmt.Errorf("invalid request_count: %d (must be at least 1)", b.ReviewerCount)
    ...
    return fmt.Errorf("invalid reviewer configuration: max_request_count %d must be at least request_count %d", b.MaxReviewerCount, b.ReviewerCount)

### [should-fix] missing regression test for repo-shard defaulting
- where: `pkg/plugins/config_test.go` (TestBlunderbussFor, TestBlunderbussMergeFrom), `pkg/plugins/blunderbuss/blunderbuss_test.go` (TestHandlePullRequestShardedConfig, TestHandleStatusShardedConfig)
- concern: No test calls `Configuration.setDefaults()` end-to-end on a config with only a repo-level shard omitting `ReviewerCount` and asserts the default (2) shows up via `BlunderbussFor()`. Every existing sharded fixture pre-populates `ReviewerCount`/`WaitForStatus` explicitly, which is exactly why the blocking bug above shipped uncaught.

### [nit] repo-key convention diverges from existing Bugzilla sharding
- where: `pkg/plugins/config.go` (`BlunderbussOrgConfig.Repos map[string]BlunderbussConfig`, keyed by full `org/repo`)
- concern: The existing Bugzilla sharding (`Bugzilla.Orgs[org].Repos`) keys repos by bare `repo`. Blunderbuss instead uses fully-qualified `org/repo` keys nested under the org key already, which is redundant, and its own test (`"uses org config with invalid repo config key"`) documents that a bare `repo` key silently falls back to the org config with no validation error — a footgun tested-around rather than fixed. Two incompatible sharding conventions now coexist in the same file.

### [nit] inconsistent receiver types on related methods
- where: `pkg/plugins/config.go:1532` vs `pkg/plugins/config.go:1751`
- concern: `validateBlunderbuss` uses a value receiver (`func (b Blunderbuss) validateBlunderbuss()`) while `compileRegexpsAndDurations` uses a pointer receiver (`func (b *Blunderbuss) compileRegexpsAndDurations()`) on the same type doing structurally similar shard-walking work. Not a bug, just inconsistent style.

### [question] mergeFrom conflict philosophy diverges from Bugzilla
- where: `pkg/plugins/config.go` `Blunderbuss.mergeFrom` vs `Bugzilla`'s supplemental-config merge
- concern: Bugzilla's merge treats any duplicate org/repo config across supplemental configs as an error regardless of content; Blunderbuss's merge instead does `reflect.DeepEqual` and only errors on genuine mismatches, silently succeeding on identical duplicates. Intentional design choice, or should it match Bugzilla's stricter model for consistency?

## Resolved

### repo-level shard mutations from descendShards are silently dropped
- resolution: `descendShards` now assigns the processed value back with `orgConfig.Repos[repo] = repoConfig`, so repo-level defaults and other mutations persist. The dedicated end-to-end regression test remains missing and is tracked under Findings.

### genyaml/populate_struct changes bundled without explanation
- resolution: Commit `fe7826a84` (`fix: populate omitempty int values in doc YAML`) adds the integer population behavior and regenerates the affected documentation, making the purpose of the generated changes explicit.

## Checked

- `BlunderbussFor` precedence logic (repo > org > global > empty) — correct, well covered by `TestBlunderbussFor` including the "org key not formatted as org/repo" edge case.
- `Blunderbuss.mergeFrom` — global/org/repo conflict detection via `reflect.DeepEqual`; test cases cover conflicting-vs-matching merges at all three levels.
- `genyaml.go` `ast.StarExpr` handling and `populate_struct.go` `reflect.Int` case — narrowly scoped, needed for pointer/int `omitempty` doc generation; unrelated fields changed in the regenerated YAML docs are expected fallout from the generator now populating previously-blank ints, not scope creep.
- `HasConfigFor` and `mergeFrom` (top-level `Configuration`) updates to include `Blunderbuss` in the equality/diff sets — consistent with how other shardable plugins are wired in.
- `descendShards` now writes processed repo config values back to the map, fixing repo-level default propagation.
- Existing global-only `blunderbuss:` configs are unaffected — config schema change is purely additive at the YAML level (the only breaking change is at the Go API level: `Blunderbuss.BlunderbussConfig.ExcludeApprovers` instead of `Blunderbuss.ExcludeApprovers`).
- Repo-level `WaitForStatus.DescriptionRe` is **not** at risk of a nil-pointer panic (correcting an earlier hypothesis from the first review pass): `WaitForStatus` is itself a pointer field, so it's copied by reference even through the buggy value-copy loop, and `handleStatus()` also nil-checks `wfs` before use. Only directly-replaced pointer fields (`ReviewerCount`) are affected by the `descendShards` bug.
- `handle()`'s `reviewerCount == nil` guard converts what could have been undefined behavior into a caught, logged error — this is why the defaulting bug manifests as a functional no-op rather than a panic/crash.

## Open questions

- Is the `org/repo`-keyed `Repos` map intentionally different from Bugzilla's bare-`repo` convention, or should it be aligned / explicitly validated?
