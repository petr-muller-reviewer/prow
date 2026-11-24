---
pr: kubernetes-sigs/prow#1085
title: "config: detect unknown fields in supplemental Prow configs"
head_sha: 09a13a0ac303fd1eb6514591281c140dd54e7842
base: main
reviewed_at: 2026-11-20T13:41:39Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-11-20T13:48:01Z
  gated_head_sha: 09a13a0ac303fd1eb6514591281c140dd54e7842
  reviewed_head_sha: 09a13a0ac303fd1eb6514591281c140dd54e7842
---

# Review

## Gate

**Decision: merge.** No gating finding remains, the PR head is unchanged since review, and the only behavior change is that `LoadStrict` now honors strict decoding for supplemental shards—the PR's intended, opt-in behavior.

### Findings disposition

- None outstanding. `REVIEW.md` has no `blocking` or `should-fix` findings; its one question about default `unknown-fields` coverage is resolved (shard validation belongs in the opt-in `unknown-fields-all`). There are no GitHub reviews, inline comments, or human issue comments with feedback.

### Merge risk

- **Behavioral (low):** `LoadStrict` (`pkg/config/config.go:1686`) now rejects unknown fields in supplemental Prow shards instead of silently ignoring them. In-tree blast radius is only `checkconfig`'s opt-in `unknown-fields-all` path (`cmd/checkconfig/main.go:372-375`); Prow controllers use permissive `Load`, so deployed instances are unaffected. Out-of-tree importers of `LoadStrict` that pass supplemental dirs would see the stricter behavior.
- **Config compatibility (low):** users running `checkconfig --warnings=unknown-fields-all` may see new failures for shards carrying unknown or non-Prow keys (shards decode into `ProwConfig`, which has no `prow_ignored` escape hatch). Opt-in check; documented shard fields only; accepted as intended. Worth mentioning in the release note for checkconfig users.
- **No API signature, flag/default, CRD, wire-format, or data-migration changes.** The one-line non-test change only forwards existing YAML options.

### Gating list

- None. Merge.

## Verdict

Approve. Passing the caller's YAML options through to supplemental shard decoding makes `LoadStrict`—and therefore checkconfig's `unknown-fields-all`—reject unknown fields in shards while `Load` keeps its permissive behavior. The tests cover both modes plus the configmap `..`/suffix filtering that must stay intact.

## What this PR does

- Threads `yamlOpts` into the `yamlToConfig` call that decodes each supplemental Prow shard in `loadConfig`.
- Makes `LoadStrict` strict about unknown fields in shard files, with the failing shard path preserved by `yamlToConfig`'s existing `error unmarshalling %s` wrapping.
- Keeps `Load` permissive, since it still passes no YAML options.
- Adds a config-package test covering strict failure, permissive load plus merge, and ignored `..`-prefixed/wrong-suffix files.
- Extends the checkconfig regression test so `unknown-fields-all` reports an unknown shard field for both the default and a custom filename suffix.

## Findings

None. The change is scoped correctly and the tests cover the relevant behavior.

## Resolved

### [question] Default `unknown-fields` still does not validate shards
- where: `cmd/checkconfig/main.go:368-384`
- resolution: Discussed with the maintainer; `unknown-fields-all` is the intended home for shard validation. Keeping the default `unknown-fields` limited to the main config avoids changing default checkconfig behavior. No change requested.
- note: Because `unknown-fields-all` is opt-in, new strictness for shards carrying extra non-Prow keys (there is no `prow_ignored` escape hatch in `ProwConfig`) is acceptable.

## Checked

- `yamlToConfig` already wraps decode failures as `error unmarshalling <path>: ...`, so the shard path reaches both `LoadStrict` and checkconfig errors; both new tests assert it.
- `Load` passes nil options, so permissive decoding and merging are unchanged; the new test confirms a known shard field still merges.
- Supplemental discovery still skips `..`-prefixed paths (configmap atomic-update symlinks) and files with the wrong suffix before decoding; the new test covers both.
- `LoadStrict` has no production callers besides checkconfig, so controller runtime behavior is unaffected.
- `go test ./pkg/config/ ./cmd/checkconfig/`, `go vet` on both packages, and `gofmt -l` on the changed files all pass.

## Open questions

- None.
