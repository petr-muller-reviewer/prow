---
pr: kubernetes-sigs/prow#947
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: af6f8ef35dec69384eaaa474045f8f88571ca284
base: main
reviewed_at: 2026-09-22T21:15:58Z
verdict: approve
---

## Verdict

approve — Dep-only AWS SDK update. All four direct modules are tagged releases with 7–12 days of soak; Prow's AWS use is limited to the S3 blob provider, and no upstream behavior change conflicts with that use.

## What this PR does

- Updates `github.com/aws/aws-sdk-go-v2` from `v1.46.0` to `v1.47.0`.
- Updates the direct `config`, `credentials`, and `service/s3` modules.
- Resolves the associated indirect AWS module versions in the main and `hack/tools` module graphs.
- Does not alter Prow source, configuration, or tests.

## Findings

None.

## Checked

- Diff classification: only `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum` changed.
- Release provenance: all four target versions are tagged releases, not pseudo-versions. `v1.47.0` is from 2026-09-09; `config/v1.33.5` and `credentials/v1.20.5` are from 2026-09-14; `service/s3/v1.113.1` is from 2026-09-11.
- Import surface: `pkg/io/providers/aws.go` is the sole production AWS SDK user; it creates the S3 client with default configuration, optional static credentials, an optional endpoint, and TLS transport settings. `pkg/io/providers/aws_test.go` is the only additional use.
- Upstream range includes generated model/endpoint updates, S3 HTTP-200 error handling, and a transfer-manager deadlock fix. Prow does not import the transfer-manager package.
- `go test ./pkg/io/providers` passed at the PR head.

## Open questions

None.
