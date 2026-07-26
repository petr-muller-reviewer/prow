---
pr: kubernetes-sigs/prow#237
title: "Add Orgs sharding for Blunderbuss"
head_sha: 4e881b4c27b4f5e11c167e52800b513b7706abc0
base: main
reviewed_at: 2026-07-26T23:25:28Z
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
---

## What this PR does

- Moves `Blunderbuss` plugin config fields (`ReviewerCount`, `MaxReviewerCount`, `ExcludeApprovers`, etc.) into a new embedded `*BlunderbussConfig` struct.
- Adds `Blunderbuss.Orgs map[string]BlunderbussOrgConfig` and `BlunderbussOrgConfig.Repos map[string]BlunderbussConfig` to allow org- and repo-scoped overrides.
- Adds `Configuration.BlunderbussFor(org, repo)` resolving precedence repo > org > global, and wires all four blunderbuss event handlers (`handlePullRequestEvent`, `handleGenericCommentEvent`, `handleStatusEvent`, plus help provider) to use it instead of the flat `pc.PluginConfig.Blunderbuss`.
- Adds `Blunderbuss.SetDefaults()`, `descendShards()`, `mergeFrom()`, and a per-shard `compileRegexpsAndDurations()` to apply defaults/validation/regex-compilation/config-merging across global+org+repo shards, mirroring the existing Bugzilla sharding pattern.
- Extends `genyaml`/`populate_struct` to handle pointer-embedded structs (`ast.StarExpr`) and `omitempty` int fields, needed because `BlunderbussConfig` is now referenced via pointer; regenerates doc YAML (incidentally populating other previously-blank `omitempty` int fields across the docs, unrelated to Blunderbuss).

Since previous review: no code changes (same head SHA). Ran a three-perspective maintainer review (code quality, maintainability, deployment risk) plus an advisor synthesis, which independently converged on the same `descendShards` bug and refined severity/blast-radius understanding.

## Findings

### [blocking] repo-level shard mutations from descendShards are silently dropped
- where: `pkg/plugins/config.go:1472-1477`
- concern: `orgConfig.Repos` is `map[string]BlunderbussConfig` (value type). The loop `for repo, repoConfig := range orgConfig.Repos { f(&repoConfig) }` mutates only the loop-local copy; the result is never written back into `orgConfig.Repos[repo]`. This breaks `SetDefaults()`: repo-level `ReviewerCount` is never defaulted to 2 when omitted. Combined with the new `handle()` change at `pkg/plugins/blunderbuss/blunderbuss.go:251-253` (`if reviewerCount == nil { return fmt.Errorf(...) }`), a repo override that omits `request_count` will error on every PR/comment/status event for that repo instead of falling back to the documented default — a silent functional regression (logged error, no reviewers requested), not a crash. Org-level configs don't have this problem because `BlunderbussOrgConfig.BlunderbussConfig` is a pointer, so mutations persist through it; only the value-typed `Repos` map is affected. Three independent review passes (code quality, maintainability, deployment risk) confirmed this by code reading.
- excerpt: |
    for repo, repoConfig := range orgConfig.Repos {
        if err := f(&repoConfig); err != nil {
            errs = append(errs, fmt.Errorf("error in blunderbuss configuration for repo %s/%s: %w", org, repo, err))
        }
    }

### [should-fix] pointer value formatted with %d in validation error messages
- where: `pkg/plugins/config.go:1490` and `pkg/plugins/config.go:1500`
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

### [nit] genyaml/populate_struct changes bundled without explanation
- where: `pkg/genyaml/genyaml.go`, `pkg/genyaml/populate_struct.go`, `pkg/config/prow-config-documented.yaml`, `pkg/plugins/plugin-config-documented.yaml`
- concern: The `ast.StarExpr` and `reflect.Int` handling are needed for this PR's pointer-typed `BlunderbussConfig`, but they also change generated docs for unrelated types (Gerrit `ratelimit`, Jenkins/Tide `max_concurrency`/`max_goroutines`, project-config team IDs, `repo_milestone.maintainers_id`, etc.). Likely correct, but bundling a generic doc-gen fix into a feature PR without mentioning it in the description makes the diff harder to review/bisect.

### [nit] inconsistent receiver types on related methods
- where: `pkg/plugins/config.go:1483` vs `pkg/plugins/config.go:1669`
- concern: `validateBlunderbuss` uses a value receiver (`func (b Blunderbuss) validateBlunderbuss()`) while `compileRegexpsAndDurations` uses a pointer receiver (`func (b *Blunderbuss) compileRegexpsAndDurations()`) on the same type doing structurally similar shard-walking work. Not a bug, just inconsistent style.

### [question] mergeFrom conflict philosophy diverges from Bugzilla
- where: `pkg/plugins/config.go` `Blunderbuss.mergeFrom` vs `Bugzilla`'s supplemental-config merge
- concern: Bugzilla's merge treats any duplicate org/repo config across supplemental configs as an error regardless of content; Blunderbuss's merge instead does `reflect.DeepEqual` and only errors on genuine mismatches, silently succeeding on identical duplicates. Intentional design choice, or should it match Bugzilla's stricter model for consistency?

## Checked

- `BlunderbussFor` precedence logic (repo > org > global > empty) — correct, well covered by `TestBlunderbussFor` including the "org key not formatted as org/repo" edge case.
- `Blunderbuss.mergeFrom` — global/org/repo conflict detection via `reflect.DeepEqual`; test cases cover conflicting-vs-matching merges at all three levels.
- `genyaml.go` `ast.StarExpr` handling and `populate_struct.go` `reflect.Int` case — narrowly scoped, needed for pointer/int `omitempty` doc generation; unrelated fields changed in the regenerated YAML docs are expected fallout from the generator now populating previously-blank ints, not scope creep.
- `HasConfigFor` and `mergeFrom` (top-level `Configuration`) updates to include `Blunderbuss` in the equality/diff sets — consistent with how other shardable plugins are wired in.
- Existing global-only `blunderbuss:` configs are unaffected — config schema change is purely additive at the YAML level (the only breaking change is at the Go API level: `Blunderbuss.BlunderbussConfig.ExcludeApprovers` instead of `Blunderbuss.ExcludeApprovers`).
- Repo-level `WaitForStatus.DescriptionRe` is **not** at risk of a nil-pointer panic (correcting an earlier hypothesis from the first review pass): `WaitForStatus` is itself a pointer field, so it's copied by reference even through the buggy value-copy loop, and `handleStatus()` also nil-checks `wfs` before use. Only directly-replaced pointer fields (`ReviewerCount`) are affected by the `descendShards` bug.
- `handle()`'s `reviewerCount == nil` guard converts what could have been undefined behavior into a caught, logged error — this is why the defaulting bug manifests as a functional no-op rather than a panic/crash.

## Open questions

- Should `descendShards` write repo-level results back into the map (e.g. `rc := orgConfig.Repos[repo]; f(&rc); orgConfig.Repos[repo] = rc`), or should `Repos` become `map[string]*BlunderbussConfig` for consistency with the pointer semantics already used at the global/org level?
- Is the `org/repo`-keyed `Repos` map intentionally different from Bugzilla's bare-`repo` convention, or should it be aligned / explicitly validated?
- Is the `pkg/genyaml` generator fix intentionally bundled with this feature, or would the author prefer to split it into its own PR/commit for reviewability?
