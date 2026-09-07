---
pr: kubernetes-sigs/prow#918
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: e9f21eb5776117b92f3346c5e87f24a42a09e93f
base: main
reviewed_at: 2026-09-04T17:11:04Z
verdict: needs-discussion
---

## Verdict

Needs discussion: the production AWS configuration, credential, and S3 modules were released about three days ago. Hold this automated bump until their five-day soak window has elapsed; no code-level regression was found.

## What this PR does

- Bumps the direct AWS SDK root module from v1.43.7 to v1.45.1.
- Bumps direct config, credentials, and S3 modules to v1.33.2, v1.20.2, and v1.110.0.
- Updates the AWS SDK's indirect dependency set in both the primary and tools Go modules.
- Does not modify Prow source, tests, configuration, or generated project code.

## Findings

### [question] Allow the Aug. 31 AWS module releases to soak
- where: `go.mod:16-18`
- concern: `config v1.33.2`, `credentials v1.20.2`, and `service/s3 v1.110.0` were tagged on 2026-08-31, roughly three days before this review. These modules configure and authenticate Prow's artifact-storage client, including custom S3-compatible endpoints and optional TLS-verification disablement in `pkg/io/providers/aws.go`. Wait until at least 2026-09-06 before merging unless a concrete need for these releases outweighs the short soak period.
- excerpt: |
    github.com/aws/aws-sdk-go-v2/config v1.33.2
    github.com/aws/aws-sdk-go-v2/credentials v1.20.2
    github.com/aws/aws-sdk-go-v2/service/s3 v1.110.0

## Checked

- Classification: dependency-only; the diff touches only `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum`.
- AWS usage is one production package, `pkg/io/providers/aws.go`, plus its unit test. It creates S3 clients using static or default credentials, optional regions/endpoints, and a configurable TLS transport.
- The upstream range includes request-content-length and credential-source middleware changes, opt-in connection read timeouts, and S3 presigned-URL checksum behavior. Prow does not construct presigned URLs or use the AWS S3 transfer manager.
- The known AWS SDK EventStream and region-validation advisories are already patched by both the previous and new S3/eventstream versions in this module graph; this bump is not a security fix.
- New versions are tagged releases, not pseudo-versions. `v1.45.1` was released 2026-08-28; it has seven days of soak.
- `git diff --check` passes.

## Open questions

- Is there a reason to take the three 2026-08-31 releases before 2026-09-06, rather than letting Dependabot refresh this group after the soak window?

