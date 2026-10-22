---
pr: kubernetes-sigs/prow#1014
title: "chore(deps): bump the prometheus group across 1 directory with 2 updates"
head_sha: d423aceaf55efda26dd98d1f12089ef8056b841c
base: main
reviewed_at: 2026-10-22T01:00:26Z
verdict: approve
---

# Review

## Verdict

Approve and take now.

The reachable `golang.org/x/net` security fixes outweigh the 11–14 day freshness window, the Prometheus changes do not touch Prow’s unused API surface, and the focused packages that import the changed direct dependencies pass.

## What this PR does

- Bumps `github.com/prometheus/client_golang` from `v1.24.1` to `v1.25.0` and `github.com/prometheus/common` from `v0.72.0` to `v0.72.1` in the root module.
- Advances `golang.org/x/net` from `v0.59.0` to `v0.61.0` and `golang.org/x/text` from `v0.42.0` to `v0.43.0` as direct companion updates.
- Moves several indirect `golang.org/x/*` and `github.com/prometheus/procfs` versions in the root and `hack/tools` modules.
- Changes only dependency manifests and lockfiles, with no project source edits.

## Findings

None.

## Checked

- **Freshness:** All four direct bumps are real tagged releases, 11–14 days old as of 2026-10-22; none is a pseudo-version.
- **`github.com/prometheus/client_golang`:** Direct in 50 files / 36 packages. `v1.25.0` raises the minimum Go version to 1.26 and changes `api/prometheus/v1`, but Prow does not import that package and already targets Go 1.26. The `promhttp` nil-URL panic fix is relevant; `AcceptedFormats` is opt-in. OSV is clean at both versions and `govulncheck` reports no Prometheus findings at base or head.
- **`github.com/prometheus/common`:** Direct in one file, `pkg/metrics/push.go:31`, using `expfmt` and `model`. The `v0.72.1` header deep-copy fix is outside Prow’s usage. OSV is clean and exposure is light.
- **`golang.org/x/net`:** Direct imports are only 3 files / 2 packages, but reachability is heavy through HTTP/2 and gRPC. Base `govulncheck` reports five called advisories—`GO-2026-6603`, `GO-2026-6610`, `GO-2026-6611`, `GO-2026-6612`, and `GO-2026-6617`—and head reports none. Entry points include `pkg/github.RoundTrip`, `pkg/jira.RoundTrip`, and gRPC serving in `cmd/gangway`.
- **`golang.org/x/text`:** Direct in 2 files / 2 packages, using `cases` and `language`. The `v0.43.0` changes are in `secure/bidirule` and `unicode/bidi`, which Prow does not exercise. OSV is clean and exposure is light.
- **Indirect tail:** `x/crypto`, `x/sys`, `x/term`, `x/tools`, and `procfs` move without introducing Prometheus findings. The imported-only `GO-2026-5932` in `x/crypto` remains unchanged.
- **Verification:** Focused `go test` passed for `./pkg/metrics/...`, `./pkg/githuboauth/...`, `./pkg/spyglass/lenses/html/...`, `./pkg/plugins/golint/suggestion/...`, and `./pkg/spyglass/lenses/links/...`.

## Open questions

None.

## Dependency followups

- **`github.com/prometheus/client_golang` `v1.24.1 → v1.25.0` — use the supported process-collector constructor**
  - category: `api-upgrade`
  - necessity: `should`
  - where: `pkg/metrics/metrics.go:59`
  - unlocked by: `github.com/prometheus/client_golang v1.25.0`
  - prompt: |
      In `kubernetes-sigs/prow`, PR #1014 bumps `github.com/prometheus/client_golang` to `v1.25.0`. Replace the deprecated `prometheus.NewProcessCollector(prometheus.ProcessCollectorOpts{})` call at `pkg/metrics/metrics.go:59` with `collectors.NewProcessCollector(collectors.ProcessCollectorOpts{})`, and remove the now-unneeded `//nolint:staticcheck` and stale comment explaining the deprecated call. Keep the controller-runtime registry unregistration semantics unchanged. Run the focused metrics tests and confirm no deprecated Prometheus constructor remains at that call site.
