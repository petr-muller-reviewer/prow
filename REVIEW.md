---
pr: kubernetes-sigs/prow#978
title: "chore(deps): bump github.com/aws/aws-sdk-go-v2/service/s3 from 1.113.1 to 1.113.2 in the aws group across 1 directory"
head_sha: 0db15779de330fa2e27cdc2f8739afbab943c26f
base: main
reviewed_at: 2026-09-27T11:26:38Z
verdict: approve
---

# Review

## Verdict

Approve.

This is a direct, dep-only S3 client update from v1.113.1 to v1.113.2. The tagged release is about six days old, so it has passed the five-day minimum soak threshold but remains within the two-week caution window. It has no OSV advisories at either version and corrects handling of S3 error payloads returned with HTTP 200. That behavior reaches Prow's S3 artifact-storage provider through gocloud, but it turns masked failures into errors rather than changing a successful-path API.

## What this PR does

- Updates the direct S3 SDK requirement in `go.mod` from v1.113.1 to v1.113.2.
- Refreshes the two matching module checksums in `go.sum`.
- Takes the SDK's final rollout of HTTP-200 embedded-error handling for applicable S3 operations.

## Findings

None.

## Resolved

None.

## Checked

- Classified the change as dep-only: `go.mod` and `go.sum` are the only changed files.
- Confirmed `github.com/aws/aws-sdk-go-v2/service/s3` is direct and imported only by `pkg/io/providers/aws.go:29`.
- Confirmed v1.113.2 is a tagged 2026-09-21 release; OSV returns no advisories for v1.113.1 or v1.113.2.
- Reviewed the upstream v1.113.2 changelog and commit range: the substantive change expands detection of embedded error responses with HTTP 200, including operations gocloud uses for Prow's S3 provider.
- Ran `go test ./pkg/io/providers` and `git diff --check` successfully. `govulncheck` is not installed.

## Open questions

None.

## Dependency followups

No opportunities identified for `github.com/aws/aws-sdk-go-v2/service/s3` v1.113.1 → v1.113.2. The only substantive release change expands HTTP-200 embedded-error detection; it adds no API, deprecation, replacement, or configurable default that Prow's sole S3-client construction site in `pkg/io/providers/aws.go` can adopt.
