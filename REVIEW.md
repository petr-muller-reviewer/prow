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

## Dependency followups

### Pin S3 checksum behavior for custom/S3-compatible endpoints

- **Category**: default-change
- **Unlocked by**: `github.com/aws/aws-sdk-go-v2/service/s3` v1.66.3 → v1.106.0 (and the root `github.com/aws/aws-sdk-go-v2` module bumped in lockstep)
- **Necessity**: should
- **Call sites**: `pkg/io/providers/aws.go:52-86` (`newS3Client`) — builds the `s3.Client` via `config.LoadDefaultConfig` + `s3.NewFromConfig`; the surrounding `S3Credentials` struct (`pkg/io/providers/providers.go`) exposes `Endpoint`, `S3ForcePathStyle`, and `Insecure`, i.e. this is purpose-built to point at arbitrary S3-compatible backends, not just AWS S3.
- **What the bump crosses**: `service/s3` v1.73.0 (2025-01-15) — "S3 client behavior is updated to always calculate a checksum by default for operations that support it (such as `PutObject` or `UploadPart`), or require it (such as `DeleteObjects`). The checksum algorithm used by default now becomes CRC32... Checksum behavior can be configured using `when_supported`/`when_required` options - in code using `RequestChecksumCalculation`... `ResponseChecksumValidation`..." followed by v1.74.1 enabling response-checksum *validation* by default. This is the well-known AWS SDK v2 default change that has broken uploads/downloads against third-party S3-compatible object stores (older MinIO/Ceph RGW, etc.) that don't correctly implement the new default checksum trailers.

Handoff prompt:

```
In kubernetes-sigs/prow, PR #812 bumped github.com/aws/aws-sdk-go-v2/service/s3 from
v1.66.3 to v1.106.0 (and the root github.com/aws/aws-sdk-go-v2 module in lockstep).
Starting at service/s3 v1.73.0, the AWS SDK v2 changed its default behavior: S3 clients
now always calculate a request checksum (CRC32) for operations that support it (e.g.
PutObject, UploadPart) and, since v1.74.1, validate response checksums by default too.
This is a well-documented compatibility break for non-AWS S3-compatible object stores
(older MinIO, Ceph RGW, etc.) that don't correctly implement the new default checksum
trailers.

pkg/io/providers/aws.go's newS3Client() builds the S3 client used by Prow's blob-storage
artifact provider via config.LoadDefaultConfig(ctx, opts...) followed by
s3.NewFromConfig(cfg, s3Opts...). The surrounding S3Credentials struct
(pkg/io/providers/providers.go) exposes Endpoint (custom BaseEndpoint), S3ForcePathStyle
(UsePathStyle), and Insecure (skip TLS verify) — i.e. this code path is explicitly built
to target arbitrary S3-compatible backends, not just AWS S3. Neither
RequestChecksumCalculation nor ResponseChecksumValidation is currently set anywhere, so
this code path silently inherited the new default behavior with this bump.

Task: in newS3Client() (pkg/io/providers/aws.go), explicitly set the pre-v1.73.0 default
behavior so existing S3-compatible deployments don't silently break:
  opts = append(opts,
      config.WithRequestChecksumCalculation(aws.RequestChecksumCalculationWhenRequired),
      config.WithResponseChecksumValidation(aws.ResponseChecksumValidationWhenRequired),
  )
(add the github.com/aws/aws-sdk-go-v2/aws import). This restores "checksum only when the
S3 API requires it" (e.g. DeleteObjects) instead of "always calculate/validate when
supported", matching the SDK's pre-v1.73.0 behavior.

Acceptance criteria:
- newS3Client() explicitly sets RequestChecksumCalculationWhenRequired and
  ResponseChecksumValidationWhenRequired via the config LoadOptions.
- pkg/io/providers/aws_test.go continues to pass; add/adjust a test assertion that the
  resulting s3.Client options carry these two values.
- `go build ./...` and `go test ./pkg/io/...` pass.

Scope guard: only touch newS3Client()'s config construction and its test. Don't touch
credential loading, TLS/endpoint/path-style handling, or any other AWS SDK call site —
this is the only place aws-sdk-go-v2 is imported in the project. Don't attempt to make
this conditional on creds.Endpoint being set; apply it unconditionally for simplicity
and predictable behavior across both AWS and S3-compatible targets.
```
