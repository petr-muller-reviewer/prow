---
pr: kubernetes-sigs/prow#1031
title: "chore(deps): bump cloud.google.com/go/cloudbuild from 1.34.0 to 1.35.0"
head_sha: 7a9bf0dd5d8b0638521c87f09821384a39cbe845
base: main
reviewed_at: 2026-10-27T20:47:27Z
verdict: approve
---

# Review

## Verdict

Approve — generated-API regeneration with additive-only proto fields, no advisories, and no behavioral change on any code path prow exercises.

The only upstream commit touching the `cloudbuild` module between the two tags adds two optional string fields (`worker_release`, `resolved_worker_release`) to `BuildOptions`, `BuildOptions.PoolOption` and `PrivatePoolV1Config.WorkerConfig`. Prow's wrapper never constructs any of those messages — callers pass a fully-formed `*cloudbuildpb.Build` into `CreateBuild` — so nothing we call changes shape or behavior. OSV has no advisories for either version, `govulncheck` is clean at both merge-base and head, and the `go.sum` diff touches only the two `cloudbuild` lines, confirming no transitive drift.

## What this PR does

- Bumps the direct dependency `cloud.google.com/go/cloudbuild` from `v1.34.0` to `v1.35.0` in `go.mod`; `go.sum` changes only the two `cloudbuild` hash lines (3 insertions / 3 deletions total, no vendor directory in the repo).
- Upstream, the only `cloudbuild` change in the range is `005d7ce4` "feat: update API sources and regenerate (#20683)", which regenerates `cloudbuildpb/cloudbuild.pb.go` with the new `worker_release` / `resolved_worker_release` fields.
- The module's own `go.mod` also moved `gax-go` `v2.23.0`→`v2.26.2`, `google.golang.org/api` `v0.287.1`→`v0.288.0`, `otel/trace` `v1.44.0`→`v1.45.0` and newer `genproto` snapshots, but prow already selects equal-or-newer versions of all of them, so the effective build list is unchanged.

## Findings

None.

## Checked

- Freshness: tagged release `cloudbuild/v1.35.0`, published `2026-10-15T17:46:48Z` — 12 days old at review time. Real tag, not a pseudo-version.
- OSV: no advisories for `v1.34.0` or `v1.35.0`.
- `govulncheck` over `./pkg/googlecloudbuild/...` at merge-base `6c4d88e3b` and at head `7a9bf0dd5`: 0 findings, 0 reachable, both exit 0.
- `go build ./pkg/googlecloudbuild/...`: passes at head.
- Usage: direct dep, one package. Imports at `pkg/googlecloudbuild/client/client.go:25-26`, `pkg/googlecloudbuild/client/client_test.go:23`, `pkg/googlecloudbuild/client/fake/fake.go:23`; nothing outside `pkg/googlecloudbuild/` imports the wrapper.
- No `BuildOptions` or `PoolOption` reference anywhere in the tree (excluding vendor), so the new fields have no call site that could be affected.
- PR CI: `pull-prow-unit-test`, `pull-prow-integration`, `pull-prow-verify-lint`, `pull-prow-image-build-test` all SUCCESS; only `tide` pending.

## Open questions

- `pkg/googlecloudbuild` has no in-repo consumers — is it maintained solely for downstream importers (e.g. test-infra), and if so should that be documented so dependabot bumps here are not mistaken for dead code?

## Dependency followups

No opportunities identified. Modules and ranges examined: `cloud.google.com/go/cloudbuild` `v1.34.0`→`v1.35.0` (direct), plus the requirement bumps inside that module's own `go.mod` (`gax-go` `v2.23.0`→`v2.26.2`, `google.golang.org/api` `v0.287.1`→`v0.288.0`, `otel/trace` `v1.44.0`→`v1.45.0`, `genproto` snapshots) — all already satisfied at equal-or-newer versions by prow's `go.mod`, so no effective transitive change reached our build list. The only changelog entry is a generated API regeneration adding optional `worker_release` / `resolved_worker_release` fields; prow constructs none of the affected messages (callers supply a complete `*cloudbuildpb.Build`), and the range contains no deprecations of symbols we call.
