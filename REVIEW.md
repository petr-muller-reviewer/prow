---
pr: kubernetes-sigs/prow#990
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: 642635dccfd6535c265f65789379cdf8880ee05e
base: main
reviewed_at: 2026-11-17T00:00:45Z
verdict: approve
refresh_log:
  - at: 2026-11-17T00:00:45Z
    old_sha: a4ddb27e6c423f9925489ee45ef889215496677e
    new_sha: 642635dccfd6535c265f65789379cdf8880ee05e
    summary: "PR rebased onto newer main and merged 2026-10-10; S3 bump advanced v1.113.4 -> v1.114.0, an additive API field; no new findings."
---

# Review

## Verdict

Approve; no AWS-specific vulnerability or compatibility blocker surfaced.

All four direct AWS SDK modules were released on Sep 24, 2026, 11 days before this review. The update includes a checksum-response fix in the active S3 dependency path. The age is the only reason to defer: if the usual two-week soak is required, wait until Oct 8; otherwise the bump looks reasonable to take now.

Refreshed on 2026-11-17: the PR was rebased onto a newer `main` and merged on 2026-10-10, after the two-week soak window had passed. The only substantive change since the previous review is the S3 module moving from `v1.113.4` to `v1.114.0`. No findings were added or resolved; the approval stands.

## What this PR does

- Updates the AWS SDK core, config, credentials, and S3 modules in the main Go manifest.
- Refreshes related AWS indirect modules in `go.mod` and `hack/tools/go.mod`, with matching sum updates.
- Changes dependency metadata only; it does not modify Prow source code.

Since previous review:

- Rebased from base `6844e439` onto `e012f288`, which brought unrelated `main` dependency bumps into the diff against the old base; the PR's own change is still the AWS group only.
- S3 advanced `v1.113.4 → v1.114.0` (release 2026-09-30) instead of `v1.113.4`; it adds an optional S3 Inventory field, `IntelligentTieringReferenceDate`, with no change to Prow's call path.
- Merged 2026-10-10 as `7f345e2f`, approved by `petr-muller`.

## Findings

No findings.

## Checked

- Classification: dep-only. Changed files are `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum`.
- Direct versions: `github.com/aws/aws-sdk-go-v2` `v1.47.0 → v1.47.1`; `/config` `v1.33.5 → v1.33.6`; `/credentials` `v1.20.5 → v1.20.6`; `/service/s3` `v1.113.2 → v1.114.0`. Core, config, and credentials are tagged Sep 24, 2026; S3 `v1.114.0` is tagged Sep 30, 2026 (17 days before the merge).
- Usage: each module is directly required and `go mod why` traces each through `pkg/io/providers`. The import surface is one production package across `pkg/io/providers/aws.go` and its test. The code loads the default credential chain or static credentials, creates an S3 client, and opens a Go Cloud S3 bucket. This is a light footprint with sensitive credential, TLS, and network handling (`pkg/io/providers/aws.go:26-92`).
- Changelog: config and credentials releases are dependency refreshes. The core SDK adds an internal AWS-chunked response decoder; Prow does not call it directly, and the SDK change adds the package and tests without a caller. S3 `v1.113.3` and `v1.113.4` changelogs say dependency update; the HTTP-200 embedded-error fix was already in base version `v1.113.2`. S3 `v1.114.0` adds only the optional `IntelligentTieringReferenceDate` inventory field.
- Relevant transitive change: `service/internal/checksum` moves `v1.11.3 → v1.11.5`. Its non-200 response checksum fix can affect supported checksummed S3 responses, including partial responses, in the S3 client path Prow uses. The remaining AWS indirect changes are patch-level refreshes; the tools manifest carries a subset as indirect dependencies.
- OSV returned no advisories for old or new versions of the four direct modules or the updated checksum module. Re-checked `service/s3@v1.114.0` and `aws-sdk-go-v2@v1.47.1` on 2026-11-17: no advisories.
- `govulncheck -format json ./pkg/io/providers` succeeded at base and head. Both runs reported the same unrelated Tekton advisory `GO-2023-1901`; neither reported findings for the bumped AWS modules. The full-repository base scan was killed for resource use (exit 137). The full head scan also reported `GO-2026-5932` in `golang.org/x/crypto` and `GO-2023-1901`; because the full base scan did not complete, their repository-wide status cannot be compared here. Neither advisory is in the AWS modules changed by this PR.
- No project-code changes were present, so there was no separate source-code review.

## Open questions

None; the PR has merged. The soak question is moot (merge landed 2026-10-10, after Oct 8).

## Dependency followups

No actionable follow-up opportunities identified. Re-checked twice on 2026-11-17 (the second pass also confirmed no `Deprecated:` marker on any symbol we call in `service/s3@v1.114.0` or `aws/transport/http@v1.47.1`) against the merged PR (merge commit `7f345e2fd755a7ae95325154d900ea6ef81e5ea2`), reading the changelog for every release in each bumped range, not just the endpoints. Examined these version ranges:

- Direct: `github.com/aws/aws-sdk-go-v2` `v1.47.0 → v1.47.1`; `/config` `v1.33.5 → v1.33.6`; `/credentials` `v1.20.5 → v1.20.6`; `/service/s3` `v1.113.2 → v1.114.0`.
- Indirect in `go.mod`: `/feature/ec2/imds` `v1.20.0 → v1.20.1`; `/internal/configsources` `v1.5.3 → v1.5.4`; `/internal/endpoints/v2` `v2.8.3 → v2.8.4`; `/internal/v4a` `v1.5.3 → v1.5.4`; `/service/internal/checksum` `v1.11.3 → v1.11.5`; `/service/internal/presigned-url` `v1.14.3 → v1.14.4`; `/service/internal/s3shared` `v1.20.3 → v1.20.4`; `/service/signin` `v1.10.0 → v1.10.1`; `/service/sso` `v1.38.0 → v1.38.1`; `/service/ssooidc` `v1.43.0 → v1.43.1`; `/service/sts` `v1.51.0 → v1.51.1`.
- `hack/tools/go.mod` repeats a subset of these as indirect dependencies; it introduces no additional AWS version ranges.

Changelog items in range that could have acted on our code:

- `service/s3` `v1.114.0`: new optional S3 Inventory field `IntelligentTieringReferenceDate`. Prow's only S3 use is the gocloud `s3blob` bucket opened from `pkg/io/providers/aws.go`; it never calls the bucket-inventory APIs or reads inventory records, so there is no call site to adopt the field.
- `service/internal/checksum` `v1.11.4`: "Bug Fix: Checksum non-200 http responses" — automatic inside the S3 response path; no exported API or call-site change.
- `service/sts` `v1.51.0`: `PackedPolicySize` deprecated, `MinimumSessionTokenSize` and token-size fields added. Prow does not import `service/sts` or call `AssumeRole`.
- `config`, `credentials`, `internal/endpoints/v2`, `internal/configsources`, `internal/v4a`, `service/internal/presigned-url`, `service/internal/s3shared`, `service/signin`, `service/sso`, `service/ssooidc`, `feature/ec2/imds`: releases in range are dependency refreshes, or `Amz-Sdk-Request`/tracing/credential-source middleware-registration refactors confined to generated clients, with no exported API we call.
- Endpoint-resolution deprecation (`EndpointResolver` → `BaseEndpoint`/`EndpointResolverV2`) is already handled: `pkg/io/providers/aws.go:88` sets `o.BaseEndpoint`, the recommended replacement.

Our AWS import surface is five symbols in `pkg/io/providers`: `config.LoadDefaultConfig`, `credentials.NewStaticCredentialsProvider`, `s3.NewFromConfig`, fields `UsePathStyle`/`HTTPClient`/`BaseEndpoint` on `s3.Options`, and `awshttp.NewBuildableClient` (`pkg/io/providers/aws.go:26-92`). None is deprecated, superseded, or given a cheaper replacement in these ranges, and no new feature has a matching call site. No candidates, so no handoff prompts.
