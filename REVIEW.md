---
pr: kubernetes-sigs/prow#902
title: "chore(deps): bump github.com/google/ko from 0.18.1 to 0.19.1 in /hack/tools"
head_sha: 00ca0969b4245e14b873439e4c8cb4513542b8cb
base: main
reviewed_at: 2026-09-01T00:01:19Z
verdict: approve
---

## What this PR does

- Dependabot bump of `github.com/google/ko` `v0.18.1 → v0.19.1` in the separate `hack/tools` go module (build tooling, not the shipped Prow binaries).
- Only `hack/tools/go.mod` and `hack/tools/go.sum` change; no project source touched. Dep-only PR.
- Drags a large sweep of transitive/indirect version bumps in `ko`'s own dependency graph (cosign v2→v3, docker/moby, sigstore family, go-openapi, otel, etc.) — normal `go mod tidy` churn from the direct bump, nothing independently notable.
- `ko` is used as a blank import in `hack/tools/tools.go:43` and built as a standalone binary by `hack/tools/prowimagebuilder/main.go:74` (`go build -o _bin/ko github.com/google/ko`), used only in the image-build pipeline.

## Dependency analysis

- **Freshness**: `v0.19.1` released 2026-06-29 (proxy.golang.org / GitHub release). ~2 months old at review time — fine, well past soak time.
- **Usage**: direct dependency of `hack/tools` only, a separate go.mod for build tooling. Not linked into any Prow runtime component. Exposure limited to the image-build pipeline (CI/release tooling), not a sensitive runtime path.
- **Changelog (v0.18.1 → v0.19.1)**: v0.19.0 added a repeatable `--ldflags` flag, fixed `defaultFlags` in `.ko.yaml` being silently ignored, and fixed missing directory headers in kodata tar archives (some strict tar implementations failed extraction) — plus routine dep bumps. v0.19.1 added one fix: skip YAML encoding when a selector filters out all documents. No CVEs/security advisories in range.
- **Exposure verdict**: light. Build-time tool only, no change to how we invoke it.
- **Take**: safe to take, already merged with no reported issues.

## Findings

None.

## Checked

- Confirmed no project source files (only `hack/tools/go.mod`/`go.sum`) changed between base `995ff1ae3` and head `00ca0969b`.
- Confirmed `ko` import surface: `hack/tools/tools.go:43` (blank import) and `hack/tools/prowimagebuilder/main.go:74` (built as `_bin/ko`).
- Confirmed release age and changelog contents for v0.19.0/v0.19.1 via GitHub releases and commit compare.

## Open questions

None.
