---
pr: kubernetes-sigs/prow#813
title: "chore(deps): bump the golang-x group across 1 directory with 5 updates"
head_sha: ce1142d8bd3e64f1bdbbb9167a35e5ff991dae9e
base: main
reviewed_at: 2026-07-28T23:45:03Z
verdict: needs-discussion
---

## Summary

Dep-only PR (dependabot group bump). Only `go.mod`/`go.sum` in root and `hack/tools/` change. No project source touched.

Bumped (root `go.mod`; `hack/tools/go.mod` mirrors except `oauth2`, already at v0.36.0 there):
- golang.org/x/net v0.54.0 → v0.57.0 (direct)
- golang.org/x/oauth2 v0.34.0 → v0.36.0 (direct)
- golang.org/x/sync v0.20.0 → v0.22.0 (direct)
- golang.org/x/text v0.37.0 → v0.40.0 (direct)
- golang.org/x/time v0.12.0 → v0.15.0 (direct)
- golang.org/x/crypto v0.52.0 → v0.54.0 (indirect, not directly imported)
- golang.org/x/mod, x/sys, x/term, x/tools (indirect, toolchain-adjacent, not directly imported)

All new releases tagged (no pseudo-versions), 21-52 days old as of review date. No CVEs in any changelog range.

## Findings

### [blocking] CI red: pull-prow-verify-lint fails, but unrelated to this diff
- where: `cmd/checkconfig/main.go:1008`, `cmd/checkconfig/main.go:1545`, `cmd/deck/configured_jobs.go:83`, `cmd/deck/tide.go:193`
- concern: `pull-prow-verify-lint` fails on golangci-lint's `modernize` analyzer (`stringscut`), flagging 4 pre-existing `strings.Split`/`SplitN(...)[0]` call sites that should use `strings.Cut`. None of these files are touched by this PR's diff, and `hack/tools/go.mod` still pins `github.com/golangci/golangci-lint/v2 v2.12.2` unchanged by this bump — confirmed via `grep golangci-lint hack/tools/go.mod` showing no version delta. This is pre-existing lint debt on `main`, surfaced here only because this PR has no source diff to attribute blame to; it will block every PR's CI (not just this one) until fixed.
- excerpt: |
    cmd/checkconfig/main.go:1008:11: stringscut: strings.SplitN call can be simplified using strings.Cut (modernize)
    		org := strings.SplitN(repo, "/", 2)[0]

    cmd/checkconfig/main.go:1545:13: stringscut: strings.Split call can be simplified using strings.Cut (modernize)
    	if org := strings.Split(repo, "/")[0]; !orgsWithInstalledApp.Has(org) {

    cmd/deck/configured_jobs.go:83:9: stringscut: strings.Split call can be simplified using strings.Cut (modernize)
    		o := strings.Split(r, "/")[0]

    cmd/deck/tide.go:193:14: stringscut: strings.Split call can be simplified using strings.Cut (modernize)
    	orgRepo := strings.Split(pool, ":")[0]

### [question] How to unblock merge
- where: n/a
- concern: Should the 4 `strings.Cut` modernizations be fixed in a separate follow-up PR (preferred, keeps this PR a pure dep bump), or folded into this PR to get CI green? Either way this PR cannot merge until `pull-prow-verify-lint` passes.

## Dependency analysis detail

- x/net: used via `net/html` (spyglass HTML lens) and `xsrftoken` (githuboauth CSRF). Changelog only touches `http2`/`quic` internals — not our import surface. Light exposure.
- x/oauth2: direct, heavy usage (deck, generic-autobumper, github client, githuboauth, prstatus, gangway/google). Changelog adds safer `CredentialsFromJSONWithType*` helpers (mitigates a credential-injection risk) but we use `googleOAuth.JWTAccessTokenSourceFromJSON`, unaffected. Safe.
- x/sync: direct, `errgroup` (bugzilla, pubsub) and `semaphore` (shardedlock, ghcache, gcs upload, sidecar censor). Changelog: `semaphore.Weighted` now panics on negative weight — defensive only, we never pass negative weights.
- x/text: direct, golint suggestion formatting + spyglass links lens. Changelog fixes (`unicode/norm` infinite loop, IDNA punycode strictness) are in packages we don't import.
- x/time: direct, only `pkg/kube/ratelimiter.go` (`rate` package). Changelog is cosmetic (`Equal` vs `==`) plus toolchain-directive bumps.
- x/crypto, x/mod, x/sys, x/term, x/tools: indirect, no direct imports found outside vendor tooling. No exposure.

## Checked
- Freshness/provenance of all 10 bumped modules via proxy.golang.org — all real tags, 21-52 days old, no security advisories in commit ranges.
- Import surface of every direct dep (`net`, `oauth2`, `sync`, `text`, `time`) across the whole repo, excluding vendor.
- Whether the PR's own dep bump changed the golangci-lint tool version (`hack/tools/go.mod`) — it did not; the lint failure predates this PR.
- pull-prow-unit-test, pull-prow-unit-test-race-detector-nonblocking, pull-prow-integration, pull-prow-image-build-test — all green.

## Open questions
- Is there tracking for the pre-existing `modernize`/`stringscut` lint debt on `main`, or should this PR (or a quick follow-up) fix it directly so CI goes green?
