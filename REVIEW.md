---
pr: kubernetes-sigs/prow#865
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: b58b89c339f3fd1bdfe26362e657b603f474a65f
base: main
reviewed_at: 2026-08-25T12:01:35Z
verdict: approve
refresh_log:
  - old_sha: 872e9b417f442ac0873fabc9967bd8cf9e5e7c50
    new_sha: b58b89c339f3fd1bdfe26362e657b603f474a65f
    summary: "PR merged; head advanced only via merge with main (contains unrelated PR #859 pubsub-v2 migration commits reachable from new head), PR's own file set unchanged (go.mod/go.sum, hack/tools/go.mod/go.sum, same 4 module versions). Approved by cblecker (2x, both before this review) and merged. No new findings."
---

## Summary

Dep-only PR. Bumps 4 direct AWS SDK for Go v2 modules (patch versions) plus ~13 lockstep indirect submodules, in `go.mod`/`go.sum` and `hack/tools/go.mod`/`go.sum`. No project source changed.

- `github.com/aws/aws-sdk-go-v2` v1.43.5 → v1.43.6
- `github.com/aws/aws-sdk-go-v2/config` v1.32.36 → v1.32.37
- `github.com/aws/aws-sdk-go-v2/credentials` v1.19.35 → v1.19.36
- `github.com/aws/aws-sdk-go-v2/service/s3` v1.107.1 → v1.107.2

Release date (all four, same commit `168c59c`): 2026-08-14, 11 days old at review time. Not a pseudo-version, real tags.

Usage: all four imported only in `pkg/io/providers/aws.go` (Prow's S3 blob-storage provider, `newS3Client`/`getS3Bucket` via `gocloud.dev/blob/s3blob`); `credentials` also referenced in `aws_test.go`.

Substantive upstream change in range: fix for S3 200-error response-body handling — original body wrapped in `io.NopCloser` so `Close()` never reached the transport, leaking connections on `CompleteMultipartUpload`; now forwards the real `Closer`. `smithy-go` bumped to v1.27.8 alongside it (root fix, restores draining body before close). Directly relevant: `s3blob` uses multipart upload for larger objects, so this is an exercised path, not dead code. No CVE/security advisory involved.

## Findings

(none)

## Since previous review
- PR merged into main. Head SHA advanced from `872e9b4` to `b58b89c` via merge with main (picks up unrelated commits from merged PR #859, "Migrate to Pub/Sub v2" — not part of this PR's own diff).
- PR's own file set unchanged: `go.mod`, `go.sum`, `hack/tools/go.mod`, `hack/tools/go.sum`, same module version bumps as previously reviewed.
- Approved by cblecker (2026-08-24T19:58:56Z and 2026-08-24T20:06:36Z, both prior to this review) and merged.
- No new comments, no new review activity since previous `reviewed_at`.

## Checked
- Diff scope confirmed dep-only (`go.mod`/`go.sum`, `hack/tools/go.mod`/`go.sum` only)
- Release freshness of all 4 direct modules via proxy.golang.org (2026-08-14, 11 days old)
- Import surface via grep: confined to `pkg/io/providers/aws.go` (+ its test)
- Upstream commit range aws/aws-sdk-go-v2 v1.43.5..v1.43.6 (shared history across config/credentials/s3 submodules): mostly API-model regen churn; one real fix (S3 200-error body-close/connection-leak, #3517) relevant to our multipart-upload usage

## Open questions
(none)

## Dependency followups

No improvement opportunities identified. The merged head `b58b89c` carries two patch releases per module (the earlier revision `872e9b4` covered by the Summary above had one), so the changelog range examined here is wider than the Summary's stated versions.

- `github.com/aws/aws-sdk-go-v2` v1.43.5 → v1.43.7 (direct)
- `github.com/aws/aws-sdk-go-v2/config` v1.32.36 → v1.32.38 (direct)
- `github.com/aws/aws-sdk-go-v2/credentials` v1.19.35 → v1.19.37 (direct)
- `github.com/aws/aws-sdk-go-v2/service/s3` v1.107.1 → v1.107.3 (direct)
- `github.com/aws/smithy-go` v1.27.7 → v1.27.8 (indirect, not imported by us)

Changelog reviewed: the 10 non-regenerated commits in `v1.43.5...v1.43.7` (shared history for the four submodules). Per commit, why nothing is actionable:

- `fe3cb6a4` close original S3 response body in 200-error handling (#3517) — behavior fix; applies to our `s3blob` multipart path without code changes.
- `283dd703` expand S3 operations checking for errors inside HTTP 200 (#3523) — more of the same behavior fix, also automatic.
- `36c81210` presigner `x-amz-checksum-*` header behavior (#3522) — we never presign; no presign client anywhere outside `pkg/io/providers/aws.go`'s plain `s3.NewFromConfig`.
- `aaa3547c` close files during directory upload/download (#3514) — `feature/s3/manager` directory helpers; `gocloud.dev@v0.40.0/blob/s3blob` uses only the single-object uploader/downloader, and `feature/s3/manager` was not bumped (stays v1.17.10), so the fix is not in this tree anyway.
- `daa38bc8` Smithy version bump, `27045a8b` tmv1 flaky test, `ce8fd411` schema-serde rollout, `c3679fbb`/`b0973f93` internal changelog/snapshot housekeeping, `12e7ccdb` endpoints model — internal or generated, no API surface for us.

No deprecations in range touch the symbols we call: `awshttp.NewBuildableClient`, `config.LoadDefaultConfig`, `config.WithCredentialsProvider`, `credentials.NewStaticCredentialsProvider`, `config.WithRegion`, `s3.NewFromConfig` with `s3.Options{UsePathStyle, HTTPClient, BaseEndpoint}`.
