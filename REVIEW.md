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

## Dependency followups

No actionable opportunities identified on 2026-10-18. Accepted: 0; skipped: 0. No handoff prompts.

PR #918 merged as `1a000594c40919068dd43a5704f279f273135d18`. Examined the dependency changes in `ffc57790ba3e94ab2081a4a5d498d8baba5db8f1..e9f21eb5776117b92f3346c5e87f24a42a09e93f`; the current default branch already contains the PR, so its merge-base with the PR head cannot recover the original bump diff.

Module ranges examined (Go; SDK-relative paths below expand under `github.com/aws/aws-sdk-go-v2/`):

- Direct: `github.com/aws/aws-sdk-go-v2` v1.43.7 → v1.45.1; `config` v1.32.38 → v1.33.2; `credentials` v1.19.37 → v1.20.2; `service/s3` v1.107.3 → v1.110.0. Root/config/credentials also change as indirect tools dependencies.
- Indirect: `aws/protocol/eventstream` v1.7.18 → v1.7.20; `feature/ec2/imds` v1.18.38 → v1.19.1; `internal/configsources` v1.4.38 → v1.5.1; `internal/endpoints/v2` v2.7.38 → v2.8.1; `internal/v4a` v1.4.39 → v1.5.1.
- Indirect: `service/internal/accept-encoding` v1.13.17 → v1.13.19; `service/internal/checksum` v1.9.31 → v1.11.1; `service/internal/presigned-url` v1.13.38 → v1.14.1; `service/internal/s3shared` v1.19.39 → v1.20.1.
- Indirect: `service/signin` v1.5.7 → v1.8.0; `service/sso` v1.33.7 → v1.36.0; `service/ssooidc` v1.38.7 → v1.41.0; `service/sts` v1.45.7 → v1.48.0; `github.com/aws/smithy-go` v1.27.8 → v1.28.1.

Changelogs, including intermediate releases, do not expose a useful migration at Prow-owned call sites:

- Prow imports only the SDK root, config, credentials, and S3 modules in `pkg/io/providers/aws.go:26-29` and its test. `LoadDefaultConfig`, `NewStaticCredentialsProvider`, `NewBuildableClient`, and `NewFromConfig` remain appropriate; there is no newly deprecated API in these calls.
- Smithy v1.28.0 deprecates `ComputeContentLength` in favor of setting content length in `SetStream`. Prow imports neither Smithy nor this middleware and has no custom AWS middleware registration to migrate. Generated-client credential-source middleware changes likewise require no Prow edits. See the [Smithy changelog](https://github.com/aws/smithy-go/blob/v1.28.1/CHANGELOG.md) and [S3 changelog](https://github.com/aws/aws-sdk-go-v2/blob/service/s3/v1.110.0/service/s3/CHANGELOG.md).
- The socket-read-timeout feature is opt-in, but S3 is explicitly exempt in the SDK's [timeout table](https://github.com/aws/aws-sdk-go-v2/blob/v1.45.1/internal/timeouts/timeouts_services.go). It does not replace the S3 transport customization at `pkg/io/providers/aws.go:78-83`; no concrete credential-fetch timeout problem was identified that would justify adopting the global flag.
- Usage clarification for the earlier Checked section: Prow does use S3 presigning indirectly, via `pkg/io/opener.go:463` and Go Cloud's `s3blob.SignedURL`. Go Cloud already calls `PresignGetObject`; the checksum-query fix applies automatically and needs no call-site change. Prow also uses the transfer manager indirectly for uploads through Go Cloud; its pinned `v0.2.3` is unchanged by this PR, so later transfer-manager download fixes are not unlocked here. See the [Go Cloud S3 driver](https://github.com/google/go-cloud/blob/v0.46.0/blob/s3blob/s3blob.go).
- Remaining transitive changes update SDK dependencies or fix behavior internally, with no new Prow-owned API usage or redundant workaround to replace.
