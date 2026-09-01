---
pr: kubernetes-sigs/prow#877
title: "chore(deps): bump gocloud.dev from 0.40.0 to 0.46.0"
head_sha: d69423cf4bd44339da13a2554cfc50d9330812ab
base: main
reviewed_at: 2026-09-01T18:24:37Z
verdict: approve
---

## Verdict

Approve. This dep-only change updates the directly used Go Cloud Development Kit after a 90-day soak. Its material `blob` changes are compatible with Prow's use, including the S3 upload-manager migration: Prow does not use `blob.Writer.As` or AWS manager types.

## What this PR does

- Updates direct dependency `gocloud.dev` from `v0.40.0` to `v0.46.0` in `go.mod`.
- Regenerates `go.sum` and the indirect module graph.
- Replaces the now-unused AWS SDK v1 and S3 manager modules with AWS SDK v2 transfermanager through Go Cloud.

## Findings

No findings.

## Checked

- Classification: dep-only (`go.mod`, `go.sum` only); `git diff --check HEAD^ HEAD` is clean.
- `v0.46.0` is a tagged release from 2026-06-02T22:21:46Z, 90 days old at review; no pseudo-version or freshness concern.
- Prow imports `gocloud.dev/blob` in five production files: `pkg/io/{opener.go,option.go,iterator.go}` and `pkg/io/providers/{providers.go,aws.go}`. It uses blob storage to read, write, list, inspect, and sign artifact objects for GCS, S3, local, and memory backends.
- The relevant upstream changes are the blob OpenTelemetry migration, S3 403-to-PermissionDenied mapping, file-bucket root-escape fix, typed error support, and v0.46 S3 upload migration from `s3/manager` to `s3/transfermanager`. Prow constructs its own `*s3.Client` in `pkg/io/providers/aws.go:54-92` and never calls the affected `blob.Writer.As` API; the breaking API change does not land in Prow.
- `go test ./pkg/io/...` passes.

## Open questions

None.
