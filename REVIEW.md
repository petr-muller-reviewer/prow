---
pr: kubernetes-sigs/prow#940
title: "chore(deps): bump the golang-x group across 1 directory with 5 updates"
head_sha: a010bf70212c92100171eb4418d9aff78797c927
base: main
reviewed_at: 2026-09-14T22:43:48Z
verdict: approve
---

## Verdict

approve — Dependency-only update with no incompatible Go-version change, no source changes, and no applicable newly fixed security advisory. The newest tagged releases have six days of soak time; the remaining direct releases are 14–26 days old.

## What this PR does

- Updates direct root dependencies: `golang.org/x/net` `v0.58.0` → `v0.59.0`, `golang.org/x/oauth2` `v0.36.0` → `v0.37.0`, `golang.org/x/sync` `v0.22.0` → `v0.23.0`, `golang.org/x/text` `v0.41.0` → `v0.42.0`, and `golang.org/x/time` `v0.15.0` → `v0.16.0`.
- Updates their indirect companions: `x/crypto` `v0.55.0` → `v0.57.0`, `x/mod` `v0.38.0` → `v0.41.0`, `x/sys` `v0.47.0` → `v0.48.0`, `x/term` `v0.45.0` → `v0.46.0`, and `x/tools` `v0.48.0` → `v0.49.0`.
- Synchronizes the same applicable versions in `hack/tools/go.mod` and both `go.sum` files.

## Findings

None.

## Checked

- Classification: only `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum` changed; no project source or generated code changed.
- Provenance/freshness: all updates are tagged Go releases. `x/net` and `x/text` were released 2026-09-08 (six days old); `x/oauth2` 2026-08-25, `x/sync` 2026-08-31, and `x/time` 2026-08-19.
- Toolchain compatibility: the updated `x/{crypto,net,oauth2,sync,sys,term,text,time,mod}` modules require Go 1.26; root `go.mod` already requires Go 1.26.4 and `hack/tools/go.mod` Go 1.26.3.
- Usage/exposure: Prow imports `x/oauth2` in ten files for GitHub/GCP authentication, `x/sync` in nine files for semaphores and errgroups, `x/net` in three files for HTML parsing and CSRF tokens, `x/text` in two packages for case/language handling, and `x/time/rate` in `pkg/kube/ratelimiter.go`. The indirect modules have no direct imports.
- Upstream changes do not alter Prow's imported `x/net` paths: the `x/net` range is HTTP/2, HTTP/3, and QUIC work. `x/oauth2` corrects the GCE universe-domain metadata path; this is compatible with Prow's ADC/JWT use. `x/sync` now rejects negative semaphore capacities; Prow's fixed capacities are positive and configured capacities are intended to be non-negative. `x/text` fixes Unicode normalization edge cases, outside Prow's `cases`/`language` use. `x/time` contains only the Go-version update.
- Vulnerability review: no advisory newly fixed by these version ranges applies to the packages Prow imports; previously disclosed advisories are already fixed by the old versions.
- `git diff --check b40678da34443e1a138659efcbbe0eee04a9c0fb..HEAD` passes.
- A focused `go test` command for packages using the five direct modules was cancelled after duplicate cold-cache compilations continued for seven minutes without output or a reported failure; it is not recorded as a passing check.

## Open questions

None.
