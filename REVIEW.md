---
pr: kubernetes-sigs/prow#828
title: "chore(deps): bump vulnerable dependencies"
head_sha: 01d8ac2444bbff706a2b6c9b00c90e8bcdc56f31
base: main
reviewed_at: 2026-08-11T08:41:36Z
verdict: approve
---

## Dependency analysis

Classification: dep + code (manifest/lockfile bumps plus a `strings.Split(...)[0]`→`strings.Cut(...)` lint-fix touching 6 source files).

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

### [nit] ad-hoc org/repo split duplicates existing helpers
- where: `pkg/plugins/config.go:1373` (also `cmd/checkconfig/main.go:1008,1545`, `cmd/deck/configured_jobs.go:83`, `cmd/deck/tide.go:193,272`)
- concern: `org, _, _ := strings.Cut(repo, "/")` re-encodes pre-existing `strings.Split(repo, "/")[0]` logic rather than reusing `pkg/config.SplitRepoName`/`NewOrgRepo`. Pre-existing duplication, not introduced by this diff; `SplitRepoName` errors on a missing `/` (different semantics) and `NewOrgRepo` returns a struct, so it's not a drop-in swap. Optional drive-by only.
- excerpt: |
    org, _, _ := strings.Cut(repo, "/")

### [nit] unrelated commits bundled in one PR
- where: PR commit structure (`f7cf72fd8` dep bump, `01d8ac244` lint refactor)
- concern: A security dependency bump is bundled with an unrelated `strings.Split`->`strings.Cut` lint-fix commit. Low real risk — either commit can still be reverted/cherry-picked independently by SHA — but splitting would keep the security-relevant history cleaner and easier to backport.

## Checked
- `go build ./...` succeeds on both modules (root and `hack/tools`).
- `go mod verify` and `go mod tidy -diff` clean in both modules.
- All 9 `strings.Cut` call sites from the lint-fix commit verified semantically equivalent to the prior `strings.Split(...)[0]`, including no-separator edge cases.
- No unsafe/vulnerable-pattern usage of the bumped APIs (go-git, grpc, cel-go, sigstore/rekor, sigstore-go, timestamp-authority, go-tuf) found in project code that the CVE fixes would affect.
- go-git and grpc release provenance confirmed via proxy.golang.org and upstream release notes/commit ranges; neither is a pseudo-version or unusually fresh release.

## Open questions
- Any objection to keeping lint-fix commits split out of future security-bump PRs, so the security-relevant diff stays minimal and easy to backport?
- Was the grpc HTTP/2 frame-flood fix (v1.82.1) the actual trigger for this PR? Worth confirming gangway doesn't need `GRPC_GO_EXPERIMENTAL_CONTROL_BUFFER_THROTTLE_LIMIT` tuned from its default (100 frames).
