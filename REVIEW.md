---
pr: kubernetes-sigs/prow#812
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: 47f845d7262b749e8facd8abeb4e176efdbfcd3a
base: main
reviewed_at: 2026-08-08T13:23:58Z
verdict: approve
---

## Summary

Dependabot group bump of `github.com/aws/aws-sdk-go-v2` family, dep-only (no project code changed):
- `github.com/aws/aws-sdk-go-v2` 1.36.3 -> 1.43.0 (direct)
- `github.com/aws/aws-sdk-go-v2/config` 1.29.14 -> 1.32.31 (direct)
- `github.com/aws/aws-sdk-go-v2/credentials` 1.17.67 -> 1.19.30 (direct)
- `github.com/aws/aws-sdk-go-v2/service/s3` 1.66.3 -> 1.106.0 (direct)

Plus indirect submodule churn moving in lockstep (`aws/protocol/eventstream`, `feature/ec2/imds`, `internal/*`, `service/internal/*`, `service/sso`, `service/ssooidc`, `service/sts`, `smithy-go`, new `service/signin`), and a matching indirect-only bump in `hack/tools/go.mod`/`go.sum` (pulled in via the ECR credential helper tool dep). Only `go.mod`/`go.sum` files changed in both modules — no vendoring (module mode), no project source.

## Dependency analysis

- **Freshness**: new release cut 2026-07-21 (all four direct bumps land on the same upstream commit `4fef345`), ~18 days before review. Past the "too fresh" window, fine to take.
- **Usage**: direct, single importer but broad blast radius. `pkg/io/providers/aws.go` (+ its test) is the only file importing `aws-sdk-go-v2/{config,credentials,service/s3,aws/transport/http}` — it builds the S3-compatible blob-storage client (credential loading via static creds or default chain, TLS config incl. an `InsecureSkipVerify` toggle, bucket construction). That code is reached transitively from `pkg/io/opener.go`, `pkg/crier/reporters/gcs/*`, `pkg/pod-utils/gcs/upload.go`, `pkg/spyglass/*`, `cmd/deck/*`, `pkg/plank/reconciler.go`, `pkg/tide/codereview.go` — i.e. most of Prow's artifact storage and UI paths, though only via one narrow entry point.
- **Changelog & exposure**: read `config`, `credentials`, `service/s3` submodule CHANGELOGs across the version ranges. No CVEs/security advisories in range. Notable entries, none of which change our default behavior:
  - `credentials` v1.19.18: login-cache files now created with mode `0600` on Unix — not exercised (we use static creds / default chain, not login-cache).
  - `config` v1.32.19: opt-in `AWS_RESTRICT_FILE_PERMISSIONS` support — no-op unless set.
  - `service/s3` v1.106.0 (the landed version): adds an option to disable clock-skew correction — opt-in, default unchanged.
  - Rest is S3 feature additions (new APIs, checksum/region support) and smithy-go perf/deserialization fixes not touching code paths `aws.go` exercises.
- **Take**: safe to bump now.

## Findings

(none — dep-only PR, no project code to review)

## Checked

- Diff scope: confirmed only `go.mod`/`go.sum` (root) and `hack/tools/go.mod`/`go.sum` changed; `hack/tools` bumps are indirect-only and consistent with the root bump (same upstream versions).
- Import surface: `grep -rln --include='*.go' '"github.com/aws/aws-sdk-go-v2' .` (excl. vendor) → only `pkg/io/providers/aws.go` and its test.
- Release provenance: `proxy.golang.org` `.info` for all four direct modules resolves to real tags (not pseudo-versions) on `github.com/aws/aws-sdk-go-v2` commit `4fef345`.
- Changelogs for `config`, `credentials`, `service/s3` submodules across the full version range — no security fixes, no behavioral changes affecting our usage (`LoadDefaultConfig`, static credentials provider, basic S3 client construction).

## Open questions

(none)
