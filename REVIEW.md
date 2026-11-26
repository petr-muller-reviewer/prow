---
pr: kubernetes-sigs/prow#1087
title: "chore(deps): bump github.com/prometheus/common from 0.73.1 to 0.74.0 in the prometheus group across 1 directory"
head_sha: 213274a829267ad8b749256a114ae16b1a99922a
base: main
reviewed_at: 2026-11-26T17:04:57Z
verdict: approve
---

# Review

## Verdict

Approve: dep-only bump with no advisories on the new versions, and the one API Prow exercises from `prometheus/common` is unchanged.

Dependabot moves `github.com/prometheus/common` from `v0.73.1` to `v0.74.0` in the main module and in `hack/tools`, dragging `github.com/pandatix/go-abnf` (0.5.3 → 0.5.5) and `go.yaml.in/yaml/v2` (2.4.4 → 2.4.5) along. Nothing outside `go.mod`/`go.sum` changes. OSV reports no advisories at either version for any of the three modules, `govulncheck -scan module` is clean for them at base and head, and the release's substantive changes (a new OpenMetrics 2.0 ABNF decoder, Prometheus text format 1.0.0 support, `config.Secret` moving to its own package) all live in code paths Prow never calls. The single direct usage site still compiles and behaves identically.

## What this PR does

- Bumps `github.com/prometheus/common` `v0.73.1 → v0.74.0` — direct in `go.mod:52`, indirect in `hack/tools/go.mod:267`.
- Bumps `github.com/pandatix/go-abnf` `v0.5.3 → v0.5.5`, pulled in by `expfmt`'s new OpenMetrics 2.0 parser.
- Bumps `go.yaml.in/yaml/v2` `v2.4.4 → v2.4.5` (fix release, no API change).
- Touches only `go.mod`, `go.sum`, `hack/tools/go.mod`, `hack/tools/go.sum` — no project source, so this is a dep-only review.

## Findings

None.

## Checked

- **Freshness**: `v0.74.0` is a real tag published 2026-11-12T10:52Z (14 days old at review time), not a pseudo-version and not yanked. `go-abnf v0.5.5` is from 2026-10-31, `go.yaml.in/yaml/v2 v2.4.5` from 2026-11-02. Only note is that `v0.74.0` is exactly at the two-week mark, so soak time is adequate rather than generous.
- **Usage surface**: `github.com/prometheus/common` is imported in exactly one file — `pkg/metrics/push.go:31-32` (`expfmt`, `model`) — the pushgateway client. Call sites: `pkg/metrics/push.go:52` `model.UTF8Validation.IsValidLabelName` (already the non-deprecated successor API), `pkg/metrics/push.go:67` `expfmt.NewEncoder` with `expfmt.NewFormat(expfmt.TypeProtoDelim)`, `pkg/metrics/push.go:89` the same format string as the Content-Type header. Light, non-sensitive exposure: encoding only, no decoding of untrusted input, no auth/TLS/exec.
- **Vulnerabilities**: OSV query at `v0.73.1` and `v0.74.0` for `prometheus/common`, and at both versions of `go-abnf` and `yaml/v2` — no advisories, so nothing fixed and nothing open. `govulncheck -scan module` from `pkg/metrics` (the module root has no Go files, which makes a root-level module scan error out) reports no findings for the bumped modules on either base or head.
- **Symbol-level `govulncheck ./...`**: identical advisories at base and head, none in bumped modules — `GO-2023-1901` (`tektoncd/pipeline v1.16.0`, reached via `init`/`DeepCopyInto` chains from `cmd/deck`, `cmd/pipeline`, `pkg/metrics`, test helpers) and `GO-2026-5932` (`x/crypto v0.58.0` openpgp, module-level only, not called). Both are pre-existing and untouched by this PR.
- **Changelog v0.73.1..v0.74.0**: new `expfmt/openmetrics_2_0_parse.go` (ABNF OM 2.0 decoder, adds the `go-abnf` dependency), Prometheus text format 1.0.0 (`FmtText_1_0_0`, new `NewDecoder`/`ResponseFormat` branches), `config.Secret` moved to `config/secret` (breaking for importers of `prometheus/common/config` — Prow imports it nowhere), `x/net`/`x/oauth2` bumps (Prow already pins `golang.org/x/net v0.62.0` and `golang.org/x/oauth2 v0.38.0` per `go.mod:65-66`, so no MVS movement), and the `yaml/v2` bump.
- **Behavior of the exercised path**: `expfmt.NewFormat(TypeProtoDelim)` and the encoder are untouched by the diff (`expfmt/expfmt.go` only adds constants and keeps `TypeTextPlain` on 0.0.4 for compatibility; `expfmt/encode.go` only extends the accepted-format lists). The new decode branches are unreachable from Prow.
- **Transitive `go-abnf v0.5.5`**: no direct imports in Prow; only the OM 2.0 parser that Prow never invokes (lookahead performance, nullable-element validation, fuzz infra).
- **Transitive `go.yaml.in/yaml/v2 v2.4.5`**: reached only via `sigs.k8s.io/yaml v1.6.0` (42 files, including `cmd/checkconfig` config parsing); the release's "YAML unmarshaler validation bypass" fix targets `UnmarshalYAML`, which Prow neither implements nor reaches through `sigs.k8s.io/yaml`'s YAML→JSON path. Harmless and mildly beneficial.
- **Build**: `go build ./pkg/metrics/... ./cmd/checkconfig/...` passes at the PR head.

## Open questions

- `v0.74.0` landed 14 days ago. Is that enough soak for you, or would you rather let it sit another week? Nothing in the diff argues for urgency.

## Dependency followups

No followup opportunities. Examined `prometheus/common v0.73.1 → v0.74.0` (every release line in the range), `pandatix/go-abnf v0.5.3 → v0.5.5`, and `go.yaml.in/yaml/v2 v2.4.4 → v2.4.5`. The range adds no `Deprecated:` markers (the only ones in `expfmt/encode.go`, `Negotiate`/`NegotiateIncludingOpenMetrics`, predate v0.73.1 and are never called by Prow), the remaining new features are decode/serve-side APIs without a Prow call site (Prow only encodes delimited protobuf for the pushgateway), and the two transitive bumps are internal or bugfix-only.
