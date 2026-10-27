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
