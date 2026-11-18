---
pr: kubernetes-sigs/prow#989
title: "chore(deps): bump github.com/prometheus/common from 0.71.0 to 0.72.0 in the prometheus group across 1 directory"
head_sha: bccd8fdeca4758fc03b3a35cde4555627638362e
base: main
reviewed_at: 2026-11-18T17:46:38Z
verdict: needs-discussion
refresh_log:
  - at: 2026-11-18T17:46:38Z
    old_sha: bccd8fdeca4758fc03b3a35cde4555627638362e
    new_sha: bccd8fdeca4758fc03b3a35cde4555627638362e
    summary: "No code changes. PR approved by petr-muller, labeled lgtm/approved, and merged on 2026-10-06T00:02:37Z; a kubevirt-bot cross-reference followed on 2026-10-09."
---
# Review

## Verdict

Needs discussion: wait for more soak time unless there is a reason to take this release now. The bump is narrow, has no dependency-specific OSV advisories, and Prow's only direct use does not appear to exercise the new behavior. The release is only about one week old, and the bump is not a security fix.

## What this PR does

- Updates `github.com/prometheus/common` from `v0.71.0` to `v0.72.0` in the main module.
- Updates the same module in `hack/tools`, where it is indirect.
- Refreshes the corresponding checksums in both `go.sum` files.
- Changes no project source code; this is a dependency-only PR.

Since previous review:
- No new commits: the head is still `bccd8fdeca4758fc03b3a35cde4555627638362e`.
- `petr-muller` approved at 2026-10-05T23:36:51Z; Prow applied `lgtm` and `approved`, then merged the PR at 2026-10-06T00:02:37Z.
- The merge resolves the "soak longer?" question below: the maintainer took `v0.72.0` without further waiting.

## Findings

No code findings.

## Checked

- The diff is limited to `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum`.
- The main module uses `github.com/prometheus/common` directly; `go mod why` traces its use to `pkg/metrics`. The tools module's indirect chain goes through golangci-lint's Prometheus linter.
- Prow's sole source import is `pkg/metrics/push.go`: `model.UTF8Validation` validates grouping labels, and `expfmt.NewEncoder` serializes gathered metrics using explicit `TypeProtoDelim` for Pushgateway.
- The `v0.72.0` release was published September 28, 2026. Its notes describe experimental OpenMetrics 2.0 encoding/validation, JSON v2 sample unmarshalling changes, and format negotiation updates. Those behaviors do not appear on Prow's explicit delimited-protobuf path. The dependency now declares Go 1.26; the main and tools modules declare Go 1.26.4 and 1.26.3 respectively. [Upstream release notes](https://github.com/prometheus/common/releases/tag/v0.72.0)
- OSV returned no advisories for `v0.71.0` or `v0.72.0`. Scoped `govulncheck ./pkg/metrics/...` completed at base and head with no finding for this module. The full base `./...` scan was killed with exit 137, so reachability was compared in the only package that imports the dependency.

## Open questions

- Answered: is there an immediate need for `v0.72.0`? The PR was approved and merged on 2026-10-06 without an explicit justification for taking it early, so the soak-time concern did not block it.

## Dependency followups

No opportunities identified. Re-run 2026-11-18 against the merged bump (merge commit `b4b2b38994859131ab14b656b584e80b325096c0`).

- Modules examined: `github.com/prometheus/common` `v0.71.0`→`v0.72.0` (direct in the main module, indirect in `hack/tools`). Both `go.sum` files changed only for this module, so no transitive version moved with the bump.
- Our usage: the only imports are `pkg/metrics/push.go:31-32` (`expfmt`, `model`); the call sites are `model.UTF8Validation.IsValidLabelName` (`pkg/metrics/push.go:52`) and `expfmt.NewFormat(expfmt.TypeProtoDelim)` for the encoder and `Content-Type` (`pkg/metrics/push.go:67,89`).
- Changelog items reviewed: OpenMetrics 2.0 histogram/gauge-histogram/summary encoding (encode-only, and Prow pushes delimited protobuf to Pushgateway); the `expfmt` parser strictness fixes (no decoding in Prow); JSON v2 unmarshalling changes and the removed `model.SampleStream.UnmarshalJSON` (no `model` decoding in Prow); the new `DefaultAcceptedFormats()`/`NegotiateAccept` negotiation helpers (no server-side negotiation in the push path); and the un-deprecation of the `Fmt*` constants (Prow already uses the still-supported `NewFormat(TypeProtoDelim)`). The only new deprecations are `expfmt.Negotiate`/`NegotiateIncludingOpenMetrics` and `model.NameValidationScheme`/`LabelName.IsValid`/`IsValidMetricName`, none of which Prow calls. The dependency's Go 1.26 requirement is already satisfied (main module Go 1.26.4, tools Go 1.26.3).

No handoff prompts.
