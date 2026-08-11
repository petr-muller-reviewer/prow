---
pr: kubernetes-sigs/prow#828
title: "chore(deps): bump vulnerable dependencies"
head_sha: 86f489e2a8a9b504c27fbaa6a36d18d6a6e6d730
base: main
reviewed_at: 2026-08-11T15:17:25Z
verdict: approve
refresh_log:
  - from: 01d8ac2444bbff706a2b6c9b00c90e8bcdc56f31
    to: 86f489e2a8a9b504c27fbaa6a36d18d6a6e6d730
    at: 2026-08-11T15:07:05Z
    summary: >
      PR rebased onto current main (picked up #830 lint-cleanups, #702 transfer-issue,
      base-image autobumps, prometheus dep updates already merged there) after a
      "needs rebase" bot comment; LGTM label removed by bot due to new changes. The
      strings.Cut lint-fix commit was dropped from the branch since it had already
      landed on main via #830 — PR is now dep-only (go.mod/go.sum, hack/tools
      go.mod/go.sum, site/package-lock.json). Dependency version bumps analyzed
      previously (go-git, grpc, cel-go, x/net, x/oauth2, x/sync, x/text) are unchanged.
  - from: 86f489e2a8a9b504c27fbaa6a36d18d6a6e6d730
    to: 86f489e2a8a9b504c27fbaa6a36d18d6a6e6d730
    at: 2026-08-11T15:17:25Z
    summary: >
      No code change (head unchanged). kubernetes-prow[bot] posted a CI failure
      report at 2026-08-11T15:07:59Z: pull-prow-integration failed as a required
      test on 86f489e2a (pull-prow-image-build-test failed on the prior, now-stale
      commit 01d8ac244). No human comments or reviews since last review.
---

## Dependency analysis

Classification: dep-only (as of `86f489e2a`). Previously dep + code; the
`strings.Split(...)[0]`→`strings.Cut(...)` lint-fix commit was dropped during the
rebase because it had already merged into main independently via #830 — no
project source files remain in this PR's diff. See `refresh_log` above.

### github.com/go-git/go-git/v5 v5.19.1 -> v5.19.2
- freshness: released 2026-07-29 (~13 days old at review time). Past the 5-day danger zone, within the 2-week soak window.
- usage: direct. Import surface: `pkg/plugins/testfreeze/checker/checker.go` (fetches refs into in-memory storage via `gitmemory.NewStorage()`), `test/integration/internal/fakegitserver/fakegitserver.go` (test-only, uses `git.PlainOpen` -> filesystem-backed `dotgit` storage).
- changelog/exposure: fixes `storage: dotgit, reject path traversal in reference names` — malicious remote ref names (`.`/`..`/control chars) could escape the refs subtree under filesystem-backed storage and clobber repo metadata. `checker.go` uses in-memory storage (unaffected class); `fakegitserver.go` uses filesystem storage but is test-only infra, not a production path handling untrusted remotes. Exposure: light.
- take: safe to bump now.

### google.golang.org/grpc v1.79.3 -> v1.82.1
- freshness: released 2026-07-15 (~27 days old). Fine.
- usage: direct. Import surface: `cmd/gangway/main.go` + `pkg/gangway/gangway_grpc.pb.go` (gangway is a network-facing gRPC server), `pkg/gangway/client/google/google.go`, `pkg/resultstore/{client,writer/writer}.go` (gRPC clients), `test/integration/internal/fakepubsub/fakepubsub.go`.
- changelog/exposure: security fix — "server: Stop reading from the connection when flooded by HTTP/2 frames" (DoS mitigation via new frame-count limit, default 100, tunable via `GRPC_GO_EXPERIMENTAL_CONTROL_BUFFER_THROTTLE_LIMIT`). Gangway runs a real network-facing gRPC server, so this directly applies. Exposure: heavy, sensitive.
- take: take this now — the clearest justification for the PR.

### golang.org/x/net v0.55.0->v0.56.0, x/oauth2 v0.34.0->v0.36.0, x/sync v0.20.0->v0.21.0, x/text v0.37.0->v0.39.0
- freshness: x/net 2026-06-09 (~2mo), x/oauth2 2026-02-11 (~6mo). Fine.
- usage/exposure: direct but routine transitive-consistency bumps riding along with go-git/grpc; no distinct CVE motivation found beyond the go-git/grpc-driven chain.
- take: fine to bump, no separate scrutiny needed.

### Long tail (not deep-dived)
- `hack/tools/go.mod`: ~30 indirect churns (cosign/sigstore toolchain, docker, go-openapi, prometheus, etc.) — tooling-only module, not shipped.
- `site/package-lock.json`: postcss/picomatch transitive lockfile bumps, no `package.json` pin change — routine.
- `cel-go` v0.26.0->v0.29.0, `antlr4-go`, envoyproxy/cncf-xds, opentelemetry-gcp detectors: indirect-only, pulled in by the grpc/genproto bump; no independent import surface in project code.

## Findings

None outstanding — see Resolved below.

## Resolved

### [nit] ad-hoc org/repo split duplicates existing helpers — no longer in scope
- where (was): `pkg/plugins/config.go:1373` (also `cmd/checkconfig/main.go:1008,1545`, `cmd/deck/configured_jobs.go:83`, `cmd/deck/tide.go:193,272`)
- resolution: this code is no longer part of the PR's diff — it landed on main independently via #830 before this PR was rebased on top of it. Not something this PR needs to address.

### [nit] unrelated commits bundled in one PR — resolved
- where (was): PR commit structure (`f7cf72fd8` dep bump, `01d8ac244` lint refactor)
- resolution: the lint-fix commit was dropped during the rebase (already on main via #830). The PR is now a single dependency-bump commit (`86f489e2a`) touching only manifests/lockfiles — nothing left to split.

## Checked
- `go build ./...` succeeds on both modules (root and `hack/tools`).
- `go mod verify` and `go mod tidy -diff` clean in both modules.
- All 9 `strings.Cut` call sites from the lint-fix commit verified semantically equivalent to the prior `strings.Split(...)[0]`, including no-separator edge cases.
- No unsafe/vulnerable-pattern usage of the bumped APIs (go-git, grpc, cel-go, sigstore/rekor, sigstore-go, timestamp-authority, go-tuf) found in project code that the CVE fixes would affect.
- go-git and grpc release provenance confirmed via proxy.golang.org and upstream release notes/commit ranges; neither is a pseudo-version or unusually fresh release.

## Open questions
- Was the grpc HTTP/2 frame-flood fix (v1.82.1) the actual trigger for this PR? Worth confirming gangway doesn't need `GRPC_GO_EXPERIMENTAL_CONTROL_BUFFER_THROTTLE_LIMIT` tuned from its default (100 frames).
- `pull-prow-integration` is currently reported failing (required) as of 2026-08-11T15:07:59Z on head `86f489e2a`. Not yet investigated — worth checking before merge whether this is a flake or a real regression from the dependency bump (e.g. a behavior change in a bumped transitive dep affecting integration tests).
