---
pr: kubernetes-sigs/prow#1072
title: "chore(deps): bump github.com/GoogleCloudPlatform/testgrid from 0.0.176 to 0.0.178"
head_sha: 566c9be2364fbd5fac75887d9ff3ebcdffea7669
base: main
reviewed_at: 2026-11-16T23:07:18Z
verdict: approve
---

# Review

## Verdict

Approve: dep-only patch bump of a direct dependency, 18 days soaked, no advisories fixed or introduced, and none of the changed TestGrid code is on a sensitive path Prow exercises.

`testgrid v0.0.176 -> v0.0.178` changes no package Prow actually parses against: `metadata`, `metadata/junit` and `config` are byte-identical, `pb/config` only gains regenerated descriptors and `reserved` ranges (no exported field changes), and the one behavioral fix (`pkg/pubsub`) is in a package Prow does not import. OSV has no advisories for either version, and `govulncheck` reports the same two pre-existing module findings at base and head. The only thing that deserves a conscious nod is that the bump lifts `go.mod`'s `go` directive from 1.26.4 to 1.26.9, because testgrid v0.0.178 declares that minimum — harmless for CI (`.go-version` is 1.27.3), but it does raise the local build floor.

## What this PR does

- Bumps the direct dependency `github.com/GoogleCloudPlatform/testgrid` from `v0.0.176` to `v0.0.178` in `go.mod`, with the matching `go.sum` hashes; no transitive requirements change.
- Also raises the module's `go` directive from `1.26.4` to `1.26.9`, which dependabot did implicitly because testgrid v0.0.178's own `go.mod` declares `go 1.26.9`.
- Touches nothing in Prow's own source, config, docs or tests.
- `go mod tidy -diff` is clean and every package importing testgrid builds.

## Findings

### [nit] Go toolchain floor raised in a PR titled as a dependency bump
- where: `go.mod:3-11`
- concern: The `go` directive moves `1.26.4 -> 1.26.9` in the same commit, and neither the title nor the commit body mentions it. It is a real constraint change for anyone building with `GOTOOLCHAIN=local` and a pre-1.26.9 toolchain, even though CI is unaffected because `.go-version` pins `1.27.3`.
- excerpt: |
    go 1.26.9
    ...
    	github.com/GoogleCloudPlatform/testgrid v0.0.178

## Checked

- Freshness: `v0.0.178` is a real immutable tag cut `2026-10-29T08:56Z` (18 days before review); `v0.0.177` was `2026-10-21`. Not a pseudo-version, no soak concern.
- OSV: no advisories for `github.com/GoogleCloudPlatform/testgrid` at `v0.0.176` or `v0.0.178`.
- `govulncheck`: module and symbol scans at merge-base and PR head find the identical set — `GO-2023-1901` (tektoncd/pipeline, reachable via `DeepCopyInto`/`Validate`) and `GO-2026-5932` (golang.org/x/crypto, imported-only). Both pre-exist the bump; nothing added or fixed.
- Changelog `v0.0.176...v0.0.178`: OWNERS/canary/prod autobump churn, three Go-toolchain bumps, "Port minor changes to upstream", "Ack unprocessable PubSub messages".
- Package diff: `metadata`, `metadata/junit` and `config` unchanged; `pb/config`/`pb/state` regenerated with comment/`reserved` edits and zero exported struct-field changes; `util/gcs` swaps `== storage.ErrObjectNotExist` for `errors.Is` and adds `Path.Get()` (additive).
- Exposure: Prow uses only `gcs.Path`/`PathForSpec` from `util/gcs` (`pkg/gcsupload/options.go:26`, `pkg/io/opener.go:42`), none of the changed helpers; the pubsub fix is unimported.
- Build: `go build ./cmd/deck ./pkg/crier/reporters/gcs/... ./pkg/gcsupload ./pkg/initupload ./pkg/io ./pkg/pod-utils/downwardapi ./pkg/resultstore ./pkg/sidecar ./pkg/spyglass/...` passes; `go mod tidy -diff` empty.

## Open questions

- Could the PR description note that this also raises the module's minimum Go version to 1.26.9, so the toolchain change is visible in the merge commit?

## Dependency followups

No improvement opportunities identified. Examined `github.com/GoogleCloudPlatform/testgrid v0.0.176 -> v0.0.178` (the only module whose version changed; the `go` directive move `1.26.4 -> 1.26.9` follows from it), reading both `v0.0.177` and `v0.0.178` release notes plus the full commit list and the source diff of the packages Prow imports (`metadata`, `metadata/junit`, `util/gcs`, `config`, `pb/config`).

- No new or changed deprecations: `metadata`/`metadata/junit`/`config` are byte-identical between the two versions, and the existing `Deprecated*` fields predate `v0.0.176`.
- Nothing to adopt from the code changes: the only behavioral fix ("Ack unprocessable PubSub messages", `pkg/pubsub`, `Nack` -> `Ack` on unparseable attributes) is in a package Prow never imports; the `util/gcs` changes are internal (`errors.Is` in TestGrid's own `Stat`/`sort` paths — Prow's equivalent at `pkg/io/opener.go:177` already uses `errors.Is`) and an additive `Path.Get()` accessor Prow has no call site for (`pkg/gcsupload/options.go:63` uses the type only via `String()`/`Bucket()`/`Object()`).
- `pb/config`/`pb/state` contain only regenerated descriptors, comment edits and new `reserved` ranges; no exported field additions or removals to migrate to.
- The remaining commits are OWNERS changes and canary/prod autobumps, plus TestGrid's own Go-toolchain bumps, which unlock nothing on our side.
