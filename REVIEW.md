---
pr: kubernetes-sigs/prow#934
title: "chore(deps): bump google.golang.org/api from 0.296.0 to 0.297.0"
head_sha: 0560f22619b146cff991705f4b3ea097d59d0ab4
base: main
reviewed_at: 2026-09-14T21:20:44Z
verdict: approve
---

## Verdict

approve — This is a narrow, direct Go dependency update. Upstream v0.297.0 only raises `google.golang.org/api`'s minimum Go version from 1.25 to 1.26; Prow already requires Go 1.26.4.

## What this PR does

- Updates direct dependency `google.golang.org/api` from `v0.296.0` to `v0.297.0` in `go.mod`.
- Replaces the corresponding two module checksums in `go.sum`.
- Takes upstream's Go 1.26 minimum-version change; it does not add code or alter Prow configuration.

## Findings

None.

## Checked

- Classification: dep-only; changed files are `go.mod` and `go.sum` only.
- `v0.297.0` is a tagged release published 2026-09-02 (about 12 days before review), not a pseudo-version.
- The upstream range `v0.296.0...v0.297.0` contains only the Go 1.26 minimum-version change and release commit; no functional, dependency-graph, or security changes were identified.
- The module is direct and imported in 9 Go files (7 production): Cloud Build, Secret Manager, storage/I/O, and test helpers. Those paths involve credentials and cloud APIs, but none of the upstream change affects their behavior.
- Prow's `go.mod` declares `go 1.26.4`, satisfying the new dependency baseline.
- `git diff --check` passes.

## Open questions

None.
