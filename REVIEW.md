---
pr: kubernetes-sigs/prow#875
title: "chore(deps): bump github.com/mattn/go-zglob from 0.0.2 to 0.0.6"
head_sha: 36dbc1080631ce3e365a335dec7d54bccae874e8
base: main
reviewed_at: 2026-09-01T18:48:12Z
verdict: approve
---

## Verdict

approve — Dep-only, direct Go-module update from `github.com/mattn/go-zglob` `v0.0.2` to `v0.0.6`; no findings.

## What this PR does

- Updates the direct `go.mod` requirement from `v0.0.2` to `v0.0.6`.
- Refreshes the two corresponding `go.sum` checksums.
- Takes upstream glob matching support for braces, character ranges, negation, escaping, and improved traversal error handling.

## Findings

None.

## Checked

- Classification: only `go.mod` and `go.sum` change; no Prow source, test, configuration, or generated-code change.
- Provenance/freshness: `v0.0.6` is the signed module-proxy tag at `df0b60a2740136eb7336cefdc92d4d87a6b90f9b`, published 2024-09-01; it had approximately 723 days of soak time when the PR merged.
- Security: OSV returns no known advisory for `github.com/mattn/go-zglob`.
- Usage: direct dependency with two production imports — `pkg/plugins/updateconfig/updateconfig.go:296` matches changed filenames to configured ConfigMap patterns; `pkg/sidecar/censor.go:168` and `pkg/sidecar/censor.go:177` match censor exclude/include patterns.
- Exposure: the sidecar call site influences secret-redaction scope, but this PR does not change Prow's use of `Match`. The upstream changes are parser/features, traversal error propagation for `Glob`, and allocation improvements; no incompatible use at either Prow call site was identified.
- PR CI: unit, integration, image-build, lint, and nonblocking race-detector checks passed.

## Open questions

None.
