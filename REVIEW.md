---
pr: kubernetes-sigs/prow#1031
title: "chore(deps): bump cloud.google.com/go/cloudbuild from 1.34.0 to 1.36.0"
head_sha: eaf5ff3eaef61510a8cd8e8658abb24040d7183f
base: main
reviewed_at: 2026-10-27T23:38:16Z
verdict: approve
refresh_log:
  - old_sha: 7a9bf0dd5d8b0638521c87f09821384a39cbe845
    new_sha: eaf5ff3eaef61510a8cd8e8658abb24040d7183f
    summary: "Updated the dependency target to v1.36.0; PR diff remains limited to go.mod and go.sum, and PR #1031 merged."
---

# Review

## Verdict

Approve — the updated PR diff only changes the selected Cloud Build module version and checksums; the v1.36.0 release notes list generated API updates, and Prow source files are unchanged.

The only upstream commit touching the `cloudbuild` module between the two tags adds two optional string fields (`worker_release`, `resolved_worker_release`) to `BuildOptions`, `BuildOptions.PoolOption` and `PrivatePoolV1Config.WorkerConfig`. Prow's wrapper never constructs any of those messages — callers pass a fully-formed `*cloudbuildpb.Build` into `CreateBuild` — so nothing we call changes shape or behavior. OSV has no advisories for either version, `govulncheck` is clean at both merge-base and head, and the `go.sum` diff touches only the two `cloudbuild` lines, confirming no transitive drift.

## What this PR does

- Bumps the direct dependency `cloud.google.com/go/cloudbuild` from `v1.34.0` to `v1.36.0` in `go.mod`; the PR diff against its updated base changes the two `cloudbuild` hash lines in `go.sum` (3 insertions / 3 deletions total, no vendor directory in the repo).
- The prior review examined `v1.34.0`→`v1.35.0`: upstream commit `005d7ce4` "feat: update API sources and regenerate (#20683)" regenerated `cloudbuildpb/cloudbuild.pb.go` with the `worker_release` / `resolved_worker_release` fields.
- The `v1.36.0` release notes list another API source regeneration (#20728); the [Cloud Build changelog](https://github.com/googleapis/google-cloud-go/blob/cloudbuild/v1.36.0/cloudbuild/CHANGES.md) records this release on 2026-10-22.
- In `v1.35.0`, the module's own `go.mod` moved `gax-go` `v2.23.0`→`v2.26.2`, `google.golang.org/api` `v0.287.1`→`v0.288.0`, `otel/trace` `v1.44.0`→`v1.45.0` and newer `genproto` snapshots; the prior review found Prow already selected equal-or-newer versions of those modules.

Since previous review:

- Dependabot replaced the reviewed `v1.35.0` target with `v1.36.0`. The old reviewed head is not an ancestor of the new head because the PR now sits on a newer base; the effective PR diff is only `go.mod` and `go.sum`, with 3 insertions and 3 deletions.
- `petr-muller` submitted an approval at 2026-10-27T23:14:33Z; `kubernetes-prow[bot]` posted the approval notification at 23:14:42Z. There were no new inline review comments. The PR merged at 2026-10-27T23:15:36Z.

## Findings

None.

## Checked

- Refresh scope at `eaf5ff3eaef61510a8cd8e8658abb24040d7183f`: inspected the effective PR diff and activity since the previous review. No Prow source files changed.
- The version-specific checks below were performed during the previous review at `7a9bf0dd5d8b0638521c87f09821384a39cbe845` against `v1.35.0`; they were not rerun for this refresh.
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

No opportunities identified. Examined `cloud.google.com/go/cloudbuild` `v1.34.0`→`v1.36.0` (direct). The v1.35 and v1.36 changelog entries are API source regenerations; the generated API additions include worker-release fields and `BuildTrigger.ApprovalConfig`. Prow's Cloud Build wrapper imports the v1 client in `pkg/googlecloudbuild/client/client.go:25-26` and passes a caller-supplied `*cloudbuildpb.Build` to `CreateBuild` at `:156-159`. No in-repo code references `BuildOptions`, `BuildTrigger`, `ApprovalConfig`, worker-release fields, `PoolOption`, or `WorkerConfig`, so these additions have no concrete call site to adopt.

The Cloud Build module's changed requirements from `v1.34.0` to `v1.36.0` are: `cloud.google.com/go/iam` `v1.11.0`→`v1.12.0`; `github.com/go-logr/logr` `v1.4.3`→`v1.4.4`; `github.com/googleapis/gax-go/v2` `v2.23.0`→`v2.26.2`; OpenTelemetry `otel`, `otel/metric`, and `otel/trace` `v1.44.0`→`v1.45.0`; `google.golang.org/api` `v0.287.1`→`v0.288.0`; and the `google.golang.org/genproto` root, `googleapis/api`, and `googleapis/rpc` snapshots advanced. Prow already selects equal-or-newer versions for all of them (`iam v1.13.0`, `go-logr v1.4.4`, `gax v2.26.2`, OTel v1.46.0, Google API v0.305.0, and newer genproto snapshots), so the bump causes no transitive version changes in Prow. The merged PR diff itself changes only the Cloud Build requirement and its two checksum entries.
