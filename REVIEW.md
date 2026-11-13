---
pr: kubernetes-sigs/prow#947
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: af6f8ef35dec69384eaaa474045f8f88571ca284
base: main
reviewed_at: 2026-11-13T17:17:25Z
verdict: approve
refresh_log:
  - at: 2026-11-13T17:17:25Z
    old_sha: af6f8ef35dec69384eaaa474045f8f88571ca284
    new_sha: af6f8ef35dec69384eaaa474045f8f88571ca284
    summary: "No code changes; PR was approved and merged at the reviewed SHA."
---

## Verdict

approve — Dep-only AWS SDK update. All four direct modules are tagged releases with 7–12 days of soak; Prow's AWS use is limited to the S3 blob provider, and no upstream behavior change conflicts with that use.

## What this PR does

- Updates `github.com/aws/aws-sdk-go-v2` from `v1.46.0` to `v1.47.0`.
- Updates the direct `config`, `credentials`, and `service/s3` modules.
- Resolves the associated indirect AWS module versions in the main and `hack/tools` module graphs.
- Does not alter Prow source, configuration, or tests.

Since previous review:

- No code changes: the PR head remained `af6f8ef35dec69384eaaa474045f8f88571ca284`.
- `petr-muller` approved at 2026-09-22T21:34:14Z; the approval notifier confirmed at 21:34:21Z.
- The PR merged at 2026-09-22T21:56:13Z without further commits, reviews, or inline comments.

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

## Dependency followups

No followup opportunities identified. No items were accepted and no handoff prompts were produced.

- Examined `github.com/aws/aws-sdk-go-v2` `v1.46.0`→`v1.47.0`, `config` `v1.33.3`→`v1.33.5`, `credentials` `v1.20.3`→`v1.20.5`, and `service/s3` `v1.111.0`→`v1.113.1`.
- The in-range changelogs contain generated model/endpoint updates, internal retry-middleware changes, an S3 Object Lock variable-retention feature, S3 HTTP-200 error-handling fixes, and routine dependency updates. None deprecates or replaces an API Prow calls.
- Prow's only AWS use is `pkg/io/providers/aws.go`, which configures an S3 client with `config.LoadDefaultConfig`, `credentials.NewStaticCredentialsProvider`, `awshttp.NewBuildableClient`, `UsePathStyle`, and `BaseEndpoint`; the new Object Lock API is not used.
- Transitive bumps (`feature/ec2/imds`, `internal/configsources`, `internal/endpoints/v2`, `internal/v4a`, `service/internal/checksum`, `service/internal/presigned-url`, `service/internal/s3shared`, `service/signin`, `service/sso`, `service/ssooidc`, and `service/sts`) are not imported directly by Prow, and no direct dependency's changelog exposes a feature requiring action.
