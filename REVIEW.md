---
pr: kubernetes-sigs/prow#1014
title: "chore(deps): bump the prometheus group across 1 directory with 2 updates"
head_sha: 58cd44195963b5a87337a5736fafbf79852ae5dc
base: main
reviewed_at: 2026-11-19T15:37:16Z
verdict: approve
refresh_log:
  - at: 2026-11-19T15:37:16Z
    old_sha: d423aceaf55efda26dd98d1f12089ef8056b841c
    new_sha: 58cd44195963b5a87337a5736fafbf79852ae5dc
    summary: "Rebase onto newer main only; dependency bumps unchanged, PR approved and merged as ac45725."
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

Since previous review (2026-11-19):

- Dependabot force-pushed `58cd441` on 2026-10-22T00:58Z, rebasing the bump onto newer `main`. The dependency content is identical: `client_golang v1.25.0`, `common v0.72.1`, `x/net v0.61.0`, `x/text v0.43.0`, `procfs v0.22.0`, `x/crypto v0.58.0` are all unchanged from the reviewed commit.
- The rebase picked up `main`'s own moves (`counterfeiter v6.14.0`, `klauspost/compress v1.20.1`), which is why `hack/tools/go.mod` shows a slightly larger diff than before; no Prow source files changed.
- The PR was approved at 2026-10-22T01:00:40Z and merged as `ac45725` at 2026-10-22T01:17:32Z. No new review comments or inline comments were posted.

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
  - category: `deprecation`
  - necessity: `should`
  - where: `pkg/metrics/metrics.go:59`
  - unlocked by: `github.com/prometheus/client_golang v1.25.0` (PR #1014, merged as `ac45725`)
  - status: still applicable at merge (`pkg/metrics/metrics.go:59` on `ac45725` still calls the deprecated constructor)
  - prompt: |
      In `kubernetes-sigs/prow`, now that PR #1014 (merged as `ac45725`) bumped `github.com/prometheus/client_golang` from `v1.24.1` to `v1.25.0`, migrate the remaining deprecated collector call in `pkg/metrics/metrics.go:59`.

      What and where: `pkg/metrics/metrics.go:59` calls `prometheus.NewProcessCollector(prometheus.ProcessCollectorOpts{})` inside `ctrlruntimemetrics.Registry.Unregister(...)`. In `v1.25.0` that constructor is documented as `Deprecated: Use collectors.NewProcessCollector instead.`

      Task: replace it with `collectors.NewProcessCollector(collectors.ProcessCollectorOpts{})`, remove the `//nolint:staticcheck` on that statement, and delete the stale sentence in the block comment claiming the call "must match what controller-runtime does so we can not change it". Controller-runtime `v0.25.2` already uses `collectors.NewProcessCollector(collectors.ProcessCollectorOpts{})` (`pkg/internal/controller/metrics/metrics.go:105`), and `collectors.NewProcessCollector` delegates to the same implementation, so the descriptors and `Unregister` behavior are unchanged — verify this while editing rather than assuming it.

      Acceptance criteria: `pkg/metrics/metrics.go` contains no `prometheus.NewProcessCollector` and no `nolint:staticcheck` for it; `go test ./pkg/metrics/...` passes; `rg 'prometheus.NewProcessCollector' --glob '*.go'` returns no non-test hits.

      Scope guard: only `pkg/metrics/metrics.go`. Do not change the metrics handler, registry selection, `NewGoCollector` unregistration, or any other collector. Leave the unrelated `//nolint:staticcheck` usage alone.

- **Examined, no actionable opportunity:** `prometheus/common v0.72.0→v0.72.1` (only the `cloneRequest` header deep-copy fix); `golang.org/x/net v0.59.0→v0.61.0` (security/bug fixes only, nothing at our `net/html` and `net/xsrftoken` call sites); `golang.org/x/text v0.42.0→v0.43.0` (`bidirule`/`bidi` performance only); `prometheus/procfs v0.21.1→v0.22.0` (not imported by Prow; new `KernelTainted`, sr-iov, and `resctrlfs` APIs have no consumer here); `golang.org/x/crypto v0.57.0→v0.58.0`, `x/sys v0.48.0→v0.49.0`, `x/term v0.46.0→v0.47.0`, `x/tools v0.50.0→v0.51.0`, and `klauspost/compress v1.19.1→v1.20.1` in `hack/tools` (indirect and never imported directly; no deprecations touching our tooling). `client_golang`'s new `testutil.GatherAndFormat` and `promhttp.HandlerOpts.AcceptedFormats` were considered and rejected: no call site where they replace hand-rolled code or deliver a concrete benefit.
