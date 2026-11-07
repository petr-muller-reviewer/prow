---
pr: kubernetes-sigs/prow#1062
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: e5132fca8fd2e20c8997491bb6bacea6383f562f
base: main
reviewed_at: 2026-11-07T21:00:00Z
verdict: approve
---

# Review

## Verdict

Approve — routine AWS SDK v2 patch bump with no behavioral delta on the code paths prow exercises.

Dep-only PR: four direct AWS modules move by patch versions (`aws-sdk-go-v2` v1.47.4→v1.47.5, `config` v1.33.11→v1.33.13, `credentials` v1.20.11→v1.20.13, `service/s3` v1.114.5→v1.114.6) plus the usual 13–14 indirect module churn in both the root and `hack/tools` modules. Releases are 8–11 days old, OSV is clean for old and new versions, `govulncheck` finds no AWS advisories (module level and symbol level, base vs. head identical), and our only S3 usage — client construction in `pkg/io/providers/aws.go` feeding `gocloud.dev/blob/s3blob` — gains no new behavior beyond upstream nil-guard and panic-to-error fixes.

## What this PR does

- Bumps the four direct AWS SDK v2 modules prow imports by one or two patch releases (`go.mod:15-18`), regenerating `go.sum`.
- Pulls the matching indirect set: `smithy-go` 1.28.6→1.28.7, `sts` 1.51.5→1.51.6, `sso` 1.38.5→1.39.1, `ssooidc` 1.43.5→1.44.1, `signin` 1.11.0→1.12.1, `feature/ec2/imds`, `eventstream`, `internal/{configsources,endpoints/v2,v4a}`, `service/internal/{accept-encoding,checksum,presigned-url,s3shared}` (`go.mod:181-195`).
- Applies the same version set inside the separate `hack/tools` module, where every AWS entry is indirect.
- Changes no project source, config, docs, or tests.

## Findings

None.

## Checked

- Classification: only `go.mod`, `go.sum`, `hack/tools/go.mod`, `hack/tools/go.sum` changed relative to `9f4d1868e` — no project code to line-review.
- Freshness: `aws-sdk-go-v2` v1.47.5 and `service/s3` v1.114.6 released 2026-10-27 (11 days old); `config` v1.33.13 and `credentials` v1.20.13 released 2026-10-30 (8 days old). All tagged releases, not pseudo-versions; past the 5-day threshold, not yet two weeks.
- Vulnerabilities: OSV returns no advisories for any of the four direct modules (or `smithy-go`) at old or new version.
- `govulncheck` v1.9.0: module-level scan at merge-base `9f4d1868e` and head `e5132fca8`, plus symbol-level scans scoped to `./pkg/io/...` on both sides. Finding sets are identical base vs. head and contain no AWS modules — only pre-existing, unrelated `GO-2023-1901` (tektoncd/pipeline) and `GO-2026-6860`/`GO-2026-6866` (golang.org/x/net).
- Usage: the entire SDK surface is two files — `pkg/io/providers/aws.go` and `pkg/io/providers/aws_test.go`. `newS3Client` uses `credentials.NewStaticCredentialsProvider` (`pkg/io/providers/aws.go:64`) and `config.LoadDefaultConfig` (`pkg/io/providers/aws.go:76`), then builds an `*s3.Client` (`pkg/io/providers/aws.go:112`) handed to `s3blob.OpenBucketV2`. All `GetObject`/`UploadObject` traffic happens inside `gocloud.dev/blob/s3blob` v0.47.0; prow never calls S3 operations directly.
- Blast radius: `providers.GetBucket` is reached from `pkg/io/opener.go:222`, and `io.NewOpener` has 12 call sites (deck, prow-controller-manager, webhook-server, checkconfig, pod-utils/gcs upload, flagutil/storage, gerrit tests) — artifact storage across components, but a single narrow code path.
- Changelog: `service/s3` v1.114.6 is dependency updates only. `config` v1.33.13 is dependency-only; v1.33.12 is the smithy-go bump. `credentials` v1.20.12 turns two panics into errors (login token missing fields; STS AssumeRole/AssumeRoleWithWebIdentity response missing credentials). Core `aws-sdk-go-v2` v1.47.5 carries the smithy-go bump, an `stscreds` nil guard, and endpoint/API-model refreshes.
- Exposure of the awsjson changes: the smithy-go 1.28.7 JSON-union deserialization fix and the 2026-10-30 nil-list/map serialization fix apply to AWS JSON protocols; S3 is restxml and the gocloud reader path uses `s3.GetObject`, so neither lands in exercised code. Schema-based serialization enabled for `sso`/`ssooidc`/`signin` only matters if the default credential chain resolves to AWS SSO, which is not prow's normal static-key or instance-role setup.
- The `transfermanager` download change in core v1.47.5 does not apply here: `feature/s3/transfermanager` stays pinned at v0.4.3 (`go.mod:183`) and gocloud only uses it for uploads.
- Build/tests: `go build ./pkg/io/...` passes, `go test ./pkg/io/providers/` passes, and `go mod tidy -diff` is clean in both the root and `hack/tools` modules, so the lockfile churn is complete and consistent.

## Open questions

None.

## Dependency followups

No opportunities identified. Range examined (PR #1062, open, head `e5132fca8`):

- `github.com/aws/aws-sdk-go-v2` v1.47.4→v1.47.5, `config` v1.33.11→v1.33.13, `credentials` v1.20.11→v1.20.13, `service/s3` v1.114.5→v1.114.6 — direct.
- `aws/smithy-go` 1.28.6→1.28.7, `sts` 1.51.5→1.51.6, `sso` 1.38.5→1.39.1, `ssooidc` 1.43.5→1.44.1, `signin` 1.11.0→1.12.1, `feature/ec2/imds` 1.20.4→1.20.5, `eventstream`, `internal/{configsources,endpoints/v2,v4a}`, `service/internal/{accept-encoding,checksum,presigned-url,s3shared}` — indirect.

Reason: the changelog windows contain only dependency-version updates and model refreshes; the only behavior changes are two panic-to-error fixes in `credentials` (no call-site change needed — prow passes static credentials or uses the default chain) and schema-based de/serialization enablement in generated service code (`sso`, `ssooidc`, `signin`) that prow does not import. No deprecations, removals, or new APIs land on the surfaces prow uses: `aws`, `aws/transport/http`, `config`, `credentials`, and `service/s3` (only client construction in `pkg/io/providers/aws.go`).
