---
pr: kubernetes-sigs/prow#918
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: e9f21eb5776117b92f3346c5e87f24a42a09e93f
base: main
reviewed_at: 2026-09-07T14:31:09Z
verdict: approve
refresh_log:
  - old_sha: e9f21eb5776117b92f3346c5e87f24a42a09e93f
    new_sha: e9f21eb5776117b92f3346c5e87f24a42a09e93f
    summary: "No code changes; recorded approval and merge after the five-day soak window elapsed."
---

## Verdict

Approve: no code changed after the review, the five-day soak window has elapsed, and the PR was merged on 2026-09-04.

## What this PR does

- Bumps the direct AWS SDK root module from v1.43.7 to v1.45.1.
- Bumps direct config, credentials, and S3 modules to v1.33.2, v1.20.2, and v1.110.0.
- Updates the AWS SDK's indirect dependency set in both the primary and tools Go modules.
- Does not modify Prow source, tests, configuration, or generated project code.

Since previous review:

- No commits or code changes were added.
- `petr-muller` approved the PR at 2026-09-04T17:13:21Z; it merged as `1a000594c40919068dd43a5704f279f273135d18` at 2026-09-04T17:32:24Z.

## Findings

None.

## Resolved

### [question] Allow the Aug. 31 AWS module releases to soak
- where: `go.mod:16-18`
- resolution: The five-day soak window elapsed on 2026-09-05/06; no subsequent AWS SDK issue or PR code change was recorded before merge.
- excerpt: |
    github.com/aws/aws-sdk-go-v2/config v1.33.2
    github.com/aws/aws-sdk-go-v2/credentials v1.20.2
    github.com/aws/aws-sdk-go-v2/service/s3 v1.110.0

## Checked

- Classification: dependency-only; the diff touches only `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum`.
- AWS usage is one production package, `pkg/io/providers/aws.go`, plus its unit test. It creates S3 clients using static or default credentials, optional regions/endpoints, and a configurable TLS transport.
- The upstream range includes request-content-length and credential-source middleware changes, opt-in connection read timeouts, and S3 presigned-URL checksum behavior. Prow does not construct presigned URLs or use the AWS S3 transfer manager.
- The known AWS SDK EventStream and region-validation advisories are already patched by both the previous and new S3/eventstream versions in this module graph; this bump is not a security fix.
- New versions are tagged releases, not pseudo-versions. The newest direct modules, released 2026-08-31, now have seven days of soak.
- `git diff --check` passes.
- No inline review comments were added. The only new issue comment was Kubernetes Prow's approval notifier, posted 2026-09-04T17:13:30Z.

## Open questions

None.
