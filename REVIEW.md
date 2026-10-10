---
pr: kubernetes-sigs/prow#990
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: a4ddb27e6c423f9925489ee45ef889215496677e
base: main
reviewed_at: 2026-10-05T23:17:05Z
verdict: approve
---

# Review

## Verdict

Approve; no AWS-specific vulnerability or compatibility blocker surfaced.

All four direct AWS SDK modules were released on Sep 24, 2026, 11 days before this review. The update includes a checksum-response fix in the active S3 dependency path. The age is the only reason to defer: if the usual two-week soak is required, wait until Oct 8; otherwise the bump looks reasonable to take now.

## What this PR does

- Updates the AWS SDK core, config, credentials, and S3 modules in the main Go manifest.
- Refreshes related AWS indirect modules in `go.mod` and `hack/tools/go.mod`, with matching sum updates.
- Changes dependency metadata only; it does not modify Prow source code.

## Findings

No findings.

## Checked

- Classification: dep-only. Changed files are `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum`.
- Direct versions: `github.com/aws/aws-sdk-go-v2` `v1.47.0 → v1.47.1`; `/config` `v1.33.5 → v1.33.6`; `/credentials` `v1.20.5 → v1.20.6`; `/service/s3` `v1.113.2 → v1.113.4`. All new versions are tagged releases dated Sep 24, 2026 (11 days old on review date).
- Usage: each module is directly required and `go mod why` traces each through `pkg/io/providers`. The import surface is one production package across `pkg/io/providers/aws.go` and its test. The code loads the default credential chain or static credentials, creates an S3 client, and opens a Go Cloud S3 bucket. This is a light footprint with sensitive credential, TLS, and network handling (`pkg/io/providers/aws.go:26-92`).
- Changelog: config and credentials releases are dependency refreshes. The core SDK adds an internal AWS-chunked response decoder; Prow does not call it directly, and the SDK change adds the package and tests without a caller. S3 `v1.113.3` and `v1.113.4` changelogs say dependency update; the HTTP-200 embedded-error fix was already in base version `v1.113.2`.
- Relevant transitive change: `service/internal/checksum` moves `v1.11.3 → v1.11.5`. Its non-200 response checksum fix can affect supported checksummed S3 responses, including partial responses, in the S3 client path Prow uses. The remaining AWS indirect changes are patch-level refreshes; the tools manifest carries a subset as indirect dependencies.
- OSV returned no advisories for old or new versions of the four direct modules or the updated checksum module.
- `govulncheck -format json ./pkg/io/providers` succeeded at base and head. Both runs reported the same unrelated Tekton advisory `GO-2023-1901`; neither reported findings for the bumped AWS modules. The full-repository base scan was killed for resource use (exit 137). The full head scan also reported `GO-2026-5932` in `golang.org/x/crypto` and `GO-2023-1901`; because the full base scan did not complete, their repository-wide status cannot be compared here. Neither advisory is in the AWS modules changed by this PR.
- No project-code changes were present, so there was no separate source-code review.

## Open questions

None for the author. The remaining decision is whether to apply the usual two-week soak and defer until Oct 8.

## Dependency followups

No actionable follow-up opportunities identified. Examined these version ranges:

- Direct: `github.com/aws/aws-sdk-go-v2` `v1.47.0 → v1.47.1`; `/config` `v1.33.5 → v1.33.6`; `/credentials` `v1.20.5 → v1.20.6`; `/service/s3` `v1.113.2 → v1.113.4`.
- Indirect in `go.mod`: `/feature/ec2/imds` `v1.20.0 → v1.20.1`; `/internal/configsources` `v1.5.3 → v1.5.4`; `/internal/endpoints/v2` `v2.8.3 → v2.8.4`; `/internal/v4a` `v1.5.3 → v1.5.4`; `/service/internal/checksum` `v1.11.3 → v1.11.5`; `/service/internal/presigned-url` `v1.14.3 → v1.14.4`; `/service/internal/s3shared` `v1.20.3 → v1.20.4`; `/service/signin` `v1.10.0 → v1.10.1`; `/service/sso` `v1.38.0 → v1.38.1`; `/service/ssooidc` `v1.43.0 → v1.43.1`; `/service/sts` `v1.51.0 → v1.51.1`.
- `hack/tools/go.mod` repeats a subset of these as indirect dependencies; it introduces no additional AWS version ranges.

The config and credentials changelogs describe dependency refreshes only. The new AWS-chunked decoder is an internal package with no Prow call site. The checksum behavior change is automatic inside the S3 response path and requires no call-site migration; the S3 HTTP-200 error fix was already present at the base version. The remaining indirect updates contain no identified deprecation, replacement API, or feature tied to Prow code.
