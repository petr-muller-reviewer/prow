---
pr: kubernetes-sigs/prow#989
title: "chore(deps): bump github.com/prometheus/common from 0.71.0 to 0.72.0 in the prometheus group across 1 directory"
head_sha: bccd8fdeca4758fc03b3a35cde4555627638362e
base: main
reviewed_at: 2026-10-05T23:28:03Z
verdict: needs-discussion
---
# Review

## Verdict

Needs discussion: wait for more soak time unless there is a reason to take this release now. The bump is narrow, has no dependency-specific OSV advisories, and Prow's only direct use does not appear to exercise the new behavior. The release is only about one week old, and the bump is not a security fix.

## What this PR does

- Updates `github.com/prometheus/common` from `v0.71.0` to `v0.72.0` in the main module.
- Updates the same module in `hack/tools`, where it is indirect.
- Refreshes the corresponding checksums in both `go.sum` files.
- Changes no project source code; this is a dependency-only PR.

## Findings

No code findings.

## Checked

- The diff is limited to `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum`.
- The main module uses `github.com/prometheus/common` directly; `go mod why` traces its use to `pkg/metrics`. The tools module's indirect chain goes through golangci-lint's Prometheus linter.
- Prow's sole source import is `pkg/metrics/push.go`: `model.UTF8Validation` validates grouping labels, and `expfmt.NewEncoder` serializes gathered metrics using explicit `TypeProtoDelim` for Pushgateway.
- The `v0.72.0` release was published September 28, 2026. Its notes describe experimental OpenMetrics 2.0 encoding/validation, JSON v2 sample unmarshalling changes, and format negotiation updates. Those behaviors do not appear on Prow's explicit delimited-protobuf path. The dependency now declares Go 1.26; the main and tools modules declare Go 1.26.4 and 1.26.3 respectively. [Upstream release notes](https://github.com/prometheus/common/releases/tag/v0.72.0)
- OSV returned no advisories for `v0.71.0` or `v0.72.0`. Scoped `govulncheck ./pkg/metrics/...` completed at base and head with no finding for this module. The full base `./...` scan was killed with exit 137, so reachability was compared in the only package that imports the dependency.

## Open questions

- Is there an immediate need for `v0.72.0`? If not, let it soak longer before merging.

## Dependency followups

No opportunities identified. Examined `github.com/prometheus/common` `v0.71.0`→`v0.72.0` and its release changes. Prow only calls `model.UTF8Validation.IsValidLabelName` and uses `expfmt.NewEncoder` with `TypeProtoDelim` in `pkg/metrics/push.go`; the new OpenMetrics 2.0, JSON v2, and negotiation changes do not apply to those call sites, and no currently used API is deprecated. The release's requirement bumps to `client_model v0.6.3`, `golang.org/x/net v0.59.0`, `golang.org/x/oauth2 v0.37.0`, `golang.org/x/sys v0.48.0`, and `golang.org/x/text v0.42.0` were already selected in both Prow modules, so no transitive module version changed in this PR. No handoff prompts.
