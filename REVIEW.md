---
pr: kubernetes-sigs/prow#1089
title: "chore(deps): bump github.com/go-git/go-git/v5 from 5.19.2 to 5.19.3"
head_sha: 569227d7241b0fa205e6dd245232461e541b3fc8
base: main
reviewed_at: 2026-11-23T17:12:10Z
verdict: approve
---

# Review

## Verdict

Approve — safe to bump now.

Dependabot-only change to `go.mod`/`go.sum`. The new release is a ~50-day-old tagged release, the module has no OSV advisories at either the old or new version, the project builds and the direct consumer's tests pass, and the manifest is tidy. Our production code only uses go-git for an anonymous `ls-remote`, so the release's fixes and hardening land mostly outside the paths we exercise.

## What this PR does

- Bumps the direct dependency `github.com/go-git/go-git/v5` from 5.19.2 to 5.19.3; only `go.mod` and `go.sum` change.
- Pulls transitively bump `github.com/go-git/go-billy/v5` 5.9.0 → 5.9.2 and `github.com/pjbgf/sha1cd` 0.6.0 → 0.7.0.
- Drops `github.com/klauspost/cpuid/v2` v2.4.0 from the module graph because sha1cd 0.7.0 replaced it with an in-repo CPU feature check.
- Requires no project-code changes: our production use is limited to anonymous remote ref listing in the testfreeze checker.

## Findings

No findings.

## Checked

- Freshness: go-git v5.19.3 tagged 2026-10-04 (~50 days old, real tag); go-billy v5.9.2 published 2026-10-02; sha1cd v0.7.0 published 2026-09-30. All are past the soak window.
- Vulnerabilities: OSV reports no advisories for go-git 5.19.2 or 5.19.3. The recent v5 advisories are already fixed at the old version (symlink/worktree and malicious-ref-name in 5.19.2, malformed-object parsing in 5.19.1), so this bump neither fixes nor leaves open any advisory.
- govulncheck: `-scan module` cannot run on this repo layout (root has no Go files, and patterns are rejected for module scanning); `-scan package ./...` on both merge-base and head produced zero findings on each side.
- Usage surface: direct import in 4 files. Production path is only `pkg/plugins/testfreeze/checker/checker.go:162` (`git.NewRemote`) and `:314-315` (`Remote.List` against `https://github.com/kubernetes/kubernetes`, no auth, memory storage). `test/integration/internal/fakegitserver/fakegitserver.go:188,214,231,243,304` is integration-test-only (`PlainInit`/`PlainOpen`/`Log`/`Config`). No `Push`, `Clone`, `Checkout`, or `Worktree` usage.
- Changelog: `config.unmarshalCore` now reads `repositoryformatversion`; `commondir` is closed (Windows cleanup leak); revlist and tree-diff walks can skip path validation; cyclic packfile delta chains are broken instead of recursed; `plumbing/transport/http/common.go` gains redirect hardening (+378/−21, rejects scheme downgrades, strips headers across origins, redacts URLs in errors); the module floor moves from Go 1.25.0 to 1.26.0. Only the HTTP transport intersects our production path, and an anonymous ls-remote has no credentials to contain.
- Compatibility: repo is on `go 1.26.9` with `.go-version` 1.27.3, so go-git's new Go 1.26.0 floor is satisfied. The x/crypto v0.56.0 floor is already exceeded here (`golang.org/x/crypto v0.58.0`, `x/net v0.62.0`, `x/text v0.44.0`).
- Manifest and build: `go mod verify` reports all modules verified, `go mod tidy -diff` is clean, `go build ./pkg/plugins/testfreeze/... ./test/integration/internal/fakegitserver/` succeeds, and `go test ./pkg/plugins/testfreeze/...` passes.
- Transitive bumps: go-billy 5.9.0 → 5.9.2 changes only its `go.mod`/`go.sum` (x/net and x/text security floors already exceeded here); sha1cd 0.7.0 replaces cpuid with internal CPU detection, copies the digest before finalizing in `CollisionResistantSum`, and adds a nil guard in `CalculateDvMask`. We do not import either module directly.

## Open questions

None.

## Dependency followups

No opportunities identified. Examined `github.com/go-git/go-git/v5` 5.19.2→5.19.3, `github.com/go-git/go-billy/v5` 5.9.0→5.9.2, `github.com/pjbgf/sha1cd` 0.6.0→0.7.0, and the removal of `github.com/klauspost/cpuid/v2`. The range contains no deprecations, and the new or changed APIs (revlist/tree-diff `SkipPathValidation`, `repositoryformatversion` parsing, HTTP redirect hardening) do not touch our call sites, which are limited to `NewRemote` + `Remote.List` in production and `PlainInit`/`PlainOpen`/`Log`/`Config` in integration test fixtures.
