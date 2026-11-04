---
pr: kubernetes-sigs/prow#1049
title: "chore(deps): bump github.com/go-test/deep from 1.1.1 to 1.1.2"
head_sha: 907d375010acb55b13b82faa61f900126fee83c7
base: main
reviewed_at: 2026-11-04T16:55:49Z
verdict: approve
refresh_log:
  - at: 2026-11-04T16:55:49Z
    from_sha: 907d375010acb55b13b82faa61f900126fee83c7
    to_sha: 907d375010acb55b13b82faa61f900126fee83c7
    summary: "No code changes; cblecker approved and the PR was merged 2026-11-04T03:02:39Z."
---

# Review

## Verdict

Safe to merge: test-only dependency, a 20-day-old tagged release, no advisories, and the one behavior change does not touch our call sites.

The PR is dep-only (`go.mod` / `go.sum`; no vendor directory, no project code). `github.com/go-test/deep` is a direct module requirement but is imported only from three `_test.go` files, so blast radius is limited to test assertions. The upstream fix in v1.1.2 (honoring `Equal()` on non-struct values) changes comparison behavior, but the values we compare with `deep.Equal` are `[]string`, `map[string]ClusterStatus`, and `*prowapi.Refs` — none implement `Equal` — so the new dispatch path is not exercised. Targeted tests at head pass.

## What this PR does

- Bumps `github.com/go-test/deep` from `v1.1.1` to `v1.1.2` in `go.mod`, with matching `go.sum` hashes; no other files change.
- Pulls in the upstream fix for issue #32 / PR #72: `Equal()` methods are now honored on non-struct values (e.g. `net.IP`, named scalars), not only on struct fields.
- Brings upstream docs/typo fixes and a CI matrix update to Go 1.24–1.27; the module's minimum Go version becomes 1.24, compatible with our `go 1.26.4`.

Since previous review:

- No code changes: the head is still `907d3750`.
- `cblecker` approved at 2026-11-04T02:08:12Z; the PR merged at 2026-11-04T03:02:39Z.
- All checks, including the previously pending required `tide` check, are green.

## Findings

No findings.

## Checked

- Freshness: `v1.1.2` is a tagged release (not a pseudo-version), published 2026-10-13T16:03:28Z via the Go module proxy — 20 days old at review time, past the soak window.
- Vulnerabilities: OSV reports no advisories for `github.com/go-test/deep` at `v1.1.1` or `v1.1.2`.
- Reachability: `govulncheck -scan module` on merge-base `c06373a3e` and head `907d375` both exit 0 with zero findings. Scans require `GOEXPERIMENT=jsonv2` with the installed Go 1.26 toolchain because `pkg/config/zz_generated.deepcopy.go` imports `encoding/json/jsontext`.
- Usage: `go mod why` chain is `sigs.k8s.io/prow/pkg/plank -> pkg/plank.test -> github.com/go-test/deep`. Imports are all in tests: `pkg/plank/reconciler_test.go:30`, `pkg/tide/status_test.go:27`, `pkg/tide/tide_test.go:38` — five `deep.Equal` call sites total.
- Changelog exposure: the only behavioral change is the `Equal()` dispatch move in `deep.go`; our compared types (`[]string`, `map[string]ClusterStatus`, `*prowapi.Refs`) have no `Equal` methods, so it is inert here. No security fixes in the range.
- Verification: `go test ./pkg/plank -run 'TestProwJobIndexer|TestSyncClusterStatus'` and `go test ./pkg/tide -run 'TestIndexFuncPassingJobs|TestDividePool|TestNonFailedBatchByBaseAndPullsIndexFunc'` pass at head; `go mod verify` reports all modules verified.
- PR state: approved and merged 2026-11-04T03:02:39Z; all checks including `tide` green.

## Open questions

None.

## Dependency followups

No improvement opportunities identified. PR #1049 is merged (merge commit `52aea4401dc419b803d912f503418c9e8a517c30`, 2026-11-04T03:02:39Z), and the analysis below still applies to the merged tree.

- Examined `github.com/go-test/deep` `v1.1.1 → v1.1.2` — the only module changed (direct requirement, test-only imports, no vendor tree).
- The v1.1.2 changelog contains only the `Equal()` dispatch fix for non-struct values (#32 / PR #72), docs/typo fixes, and a Go 1.24–1.27 CI matrix update. No deprecations, no new APIs, no default changes.
- Our five `deep.Equal` call sites (`pkg/plank/reconciler_test.go:389`, `:755`, `pkg/tide/status_test.go:2219`, `pkg/tide/tide_test.go:1151`, `:4660`) compare `[]string`, `map[string]ClusterStatus`, and `*prowapi.Refs`; none of those types implement `Equal`, so there is no call site to migrate to the new behavior.
