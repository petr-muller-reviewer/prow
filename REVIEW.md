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

## Dependency followups

Examined 2026-10-19: no actionable opportunities identified; no items or handoff prompts recorded.

- Module examined: direct Go dependency `github.com/mattn/go-zglob` `v0.0.2 → v0.0.6`, including intermediate releases `v0.0.3`, `v0.0.4`, and `v0.0.5`. No transitive modules changed; the new module manifest has no dependency requirements.
- Context: PR #875 merged as `382c662413df5dcac02778c55b09291f3bae1da6`. Compared its original base `fc981531d74bb22d73916199e35e944b8c09fc38` with PR head `36dbc1080631ce3e365a335dec7d54bccae874e8`; the current tracking branch already contains the PR, and local HEAD additionally contains the saved review commit.
- Sources: [upstream commit and source comparison](https://github.com/mattn/go-zglob/compare/v0.0.2...v0.0.6) and [v0.0.6 module manifest](https://github.com/mattn/go-zglob/blob/v0.0.6/go.mod). Upstream has no GitHub release notes or changelog file at this tag.
- Usage examined: `pkg/plugins/updateconfig/updateconfig.go:296` and `pkg/sidecar/censor.go:168`, `pkg/sidecar/censor.go:177`, plus their tests and configuration documentation. All three calls use `Match`; no deprecated API or improved replacement requires migration.
- Reason: brace handling, ranges, negation, and escaping are available through existing configuration patterns without adapting Prow's callers; no concrete pattern duplication or hand-written matching workaround was identified that these features would remove. Existing brace-alternative coverage at `pkg/plugins/updateconfig/updateconfig_test.go:1122` already uses `prow/*.{yaml,yml}`. Traversal improvements concern `Glob`, which these callers do not use, and allocation improvements apply automatically inside the dependency.
