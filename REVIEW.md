---
pr: kubernetes-sigs/prow#812
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: 47f845d7262b749e8facd8abeb4e176efdbfcd3a
base: main
reviewed_at: 2026-10-09T13:27:38Z
verdict: approve
refresh_log:
  - old_sha: 47f845d7262b749e8facd8abeb4e176efdbfcd3a
    new_sha: 47f845d7262b749e8facd8abeb4e176efdbfcd3a
    summary: "No code changes; recorded approval, bot notification, and merge."
---

## Summary

Dependabot group bump of `github.com/aws/aws-sdk-go-v2` family, dep-only (no project code changed):
- `github.com/aws/aws-sdk-go-v2` 1.36.3 -> 1.43.0 (direct)
- `github.com/aws/aws-sdk-go-v2/config` 1.29.14 -> 1.32.31 (direct)
- `github.com/aws/aws-sdk-go-v2/credentials` 1.17.67 -> 1.19.30 (direct)
- `github.com/aws/aws-sdk-go-v2/service/s3` 1.66.3 -> 1.106.0 (direct)

Plus indirect submodule churn moving in lockstep (`aws/protocol/eventstream`, `feature/ec2/imds`, `internal/*`, `service/internal/*`, `service/sso`, `service/ssooidc`, `service/sts`, `smithy-go`, new `service/signin`), and a matching indirect-only bump in `hack/tools/go.mod`/`go.sum` (pulled in via the ECR credential helper tool dep). Only `go.mod`/`go.sum` files changed in both modules — no vendoring (module mode), no project source.

**Since previous review:**

- No commits or diff changes: the PR head remains `47f845d7262b749e8facd8abeb4e176efdbfcd3a`.
- `petr-muller` submitted an approval at 2026-08-08T16:27:52Z; `kubernetes-prow[bot]` posted the approval notification at 2026-08-08T16:28:01Z.
- The PR was merged at 2026-08-08T16:49:58Z (merge commit `d06078952878afaf40df8696be3199ecc85e4837`).

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

Retained the one previously recorded followup; no additional candidates were identified on this rerun. No items were newly accepted or skipped. PR #812 merged at 2026-08-08T16:49:58Z as `d06078952878afaf40df8696be3199ecc85e4837`; carry out the handoff on the current default branch.

### Scope examined

The current `upstream/main` already contains the PR, so `git merge-base HEAD upstream/main` resolves to the PR head itself. The dependency inventory therefore uses the original single-commit PR range, `47f845d7262b749e8facd8abeb4e176efdbfcd3a^..47f845d7262b749e8facd8abeb4e176efdbfcd3a`, excluding the local review-artifact commit.

Read the full relevant release ranges for the SDK root, config, credentials, and S3, plus the checksum implementation exposed through the S3 changes:

- [SDK root changelog](https://github.com/aws/aws-sdk-go-v2/blob/v1.43.0/CHANGELOG.md): v1.36.3 → v1.43.0.
- [Config changelog](https://github.com/aws/aws-sdk-go-v2/blob/v1.43.0/config/CHANGELOG.md): v1.29.14 → v1.32.31.
- [Credentials changelog](https://github.com/aws/aws-sdk-go-v2/blob/v1.43.0/credentials/CHANGELOG.md): v1.17.67 → v1.19.30.
- [S3 changelog](https://github.com/aws/aws-sdk-go-v2/blob/v1.43.0/service/s3/CHANGELOG.md): v1.66.3 → v1.106.0.
- [Internal checksum changelog](https://github.com/aws/aws-sdk-go-v2/blob/v1.43.0/service/internal/checksum/CHANGELOG.md): v1.4.4 → v1.9.24.

Only `pkg/io/providers/aws.go` and `aws_test.go` import the changed AWS modules directly. Other indirect modules below have no owned import sites and were deprioritized, rather than treated as fully reviewed changelogs. The SDK's smithy-go release notes concern document/CBOR serialization and endpoint validation; Prow has no smithy-go call sites to migrate.

No additional actionable change was found: Prow does not call the deprecated `http.AddResponseReadTimeoutMiddleware`; login credential support works through the existing default credential chain; HTTP interceptors and service-option callbacks do not replace any owned implementation; and new S3 administrative APIs are outside the blob adapter's usage. The HTTP connection-limit change and checksum caching apply within the SDK without an application change.

### Module inventory

| Module | Root module | Tools module | Dependency type |
| --- | --- | --- | --- |
| `github.com/aws/aws-sdk-go-v2` | v1.36.3 → v1.43.0 | v1.41.4 → v1.43.0 | direct in root; indirect in tools |
| `github.com/aws/aws-sdk-go-v2/aws/protocol/eventstream` | v1.6.6 → v1.7.14 | — | indirect |
| `github.com/aws/aws-sdk-go-v2/config` | v1.29.14 → v1.32.31 | v1.32.12 → v1.32.31 | direct in root; indirect in tools |
| `github.com/aws/aws-sdk-go-v2/credentials` | v1.17.67 → v1.19.30 | v1.19.12 → v1.19.30 | direct in root; indirect in tools |
| `github.com/aws/aws-sdk-go-v2/feature/ec2/imds` | v1.16.30 → v1.18.31 | v1.18.20 → v1.18.31 | indirect |
| `github.com/aws/aws-sdk-go-v2/internal/configsources` | v1.3.34 → v1.4.31 | v1.4.20 → v1.4.31 | indirect |
| `github.com/aws/aws-sdk-go-v2/internal/endpoints/v2` | v2.6.34 → v2.7.31 | v2.7.20 → v2.7.31 | indirect |
| `github.com/aws/aws-sdk-go-v2/internal/ini` | v1.8.3 → removed | v1.8.6 → removed | indirect |
| `github.com/aws/aws-sdk-go-v2/internal/v4a` | v1.3.23 → v1.4.32 | absent → v1.4.32 | indirect |
| `github.com/aws/aws-sdk-go-v2/service/internal/accept-encoding` | v1.12.3 → v1.13.13 | v1.13.7 → v1.13.13 | indirect |
| `github.com/aws/aws-sdk-go-v2/service/internal/checksum` | v1.4.4 → v1.9.24 | — | indirect |
| `github.com/aws/aws-sdk-go-v2/service/internal/presigned-url` | v1.12.15 → v1.13.31 | v1.13.20 → v1.13.31 | indirect |
| `github.com/aws/aws-sdk-go-v2/service/internal/s3shared` | v1.18.4 → v1.19.32 | — | indirect |
| `github.com/aws/aws-sdk-go-v2/service/s3` | v1.66.3 → v1.106.0 | — | direct in root |
| `github.com/aws/aws-sdk-go-v2/service/signin` | absent → v1.5.0 | v1.0.8 → v1.5.0 | indirect |
| `github.com/aws/aws-sdk-go-v2/service/sso` | v1.25.3 → v1.33.0 | v1.30.13 → v1.33.0 | indirect |
| `github.com/aws/aws-sdk-go-v2/service/ssooidc` | v1.30.1 → v1.38.0 | v1.35.17 → v1.38.0 | indirect |
| `github.com/aws/aws-sdk-go-v2/service/sts` | v1.33.19 → v1.45.0 | v1.41.9 → v1.45.0 | indirect |
| `github.com/aws/smithy-go` | v1.22.2 → v1.27.3 | v1.24.2 → v1.27.3 | indirect |

### Pin S3 checksum behavior for custom/S3-compatible endpoints

- **Category**: default-change.
- **Unlocked by**: `github.com/aws/aws-sdk-go-v2/service/s3` v1.66.3 → v1.106.0, crossing v1.73.0; `service/internal/checksum` v1.4.4 → v1.9.24 crosses the matching implementation change in v1.5.0. The configuration helpers already existed in the old config version; the S3 default change makes an explicit policy worth considering.
- **Necessity**: should — preserve predictable behavior for deployments using custom S3-compatible endpoints.
- **Call sites**: `pkg/io/providers/aws.go:54` (`newS3Client`) and `aws.go:47` (`s3blob.OpenBucketV2`); `pkg/io/providers/providers.go:94` exposes custom endpoints. Extend `pkg/io/providers/aws_test.go:29` for the chosen policy.
- **Changelog evidence**: S3 v1.73.0 says, “S3 client behavior is updated to always calculate a checksum by default for operations that support it”. Its remaining notes describe default CRC32 request checksums and configurable response validation; v1.74.1 enables the request's checksum-validation mode by default. [Tagged changelog](https://github.com/aws/aws-sdk-go-v2/blob/v1.43.0/service/s3/CHANGELOG.md#v1730-2025-01-15).
- **Usage evidence**: `newS3Client` sets neither checksum policy. The [Go Cloud v0.40.0 adapter](https://github.com/google/go-cloud/blob/v0.40.0/blob/s3blob/s3blob.go) uses the supplied client for downloads and multipart uploads without setting checksum modes or an algorithm. A read-only check of GitHub's current default-branch `aws.go` also found no checksum policy.
- **Practical limit**: this is a compatibility risk inferred from the SDK default change and Prow's custom-endpoint support, not a reproduced failure against a Prow deployment. Do not assert that all MinIO or Ceph versions fail.
- **Policy retained from the existing handoff**: apply `WhenRequired` to both settings unconditionally. This reduces optional integrity checking for AWS S3 as well as custom endpoints, and explicit config options take precedence over environment/shared-config values. This rerun preserves the recorded policy; it does not establish that every deployment needs it. [AWS setting semantics](https://docs.aws.amazon.com/sdkref/latest/guide/feature-dataintegrity.html).

Handoff prompt:

```text
In kubernetes-sigs/prow, follow up on PR #812, "chore(deps): bump the aws group
across 1 directory with 4 updates", merged as
d06078952878afaf40df8696be3199ecc85e4837 on 2026-08-08.

Work in a fresh checkout/worktree of the current default branch. First check
whether this followup has already been implemented there.

The PR bumped github.com/aws/aws-sdk-go-v2/service/s3 from v1.66.3 to v1.106.0
and service/internal/checksum from v1.4.4 to v1.9.24. S3 v1.73.0 changed request
checksum calculation and response validation defaults to when-supported;
v1.74.1 enables the request's checksum-validation mode by default.
Source:
https://github.com/aws/aws-sdk-go-v2/blob/v1.43.0/service/s3/CHANGELOG.md#v1730-2025-01-15

Prow's pkg/io/providers/aws.go:newS3Client constructs the client passed to
gocloud.dev/blob/s3blob.OpenBucketV2. S3Credentials in providers.go supports
custom endpoints. Neither checksum setting is explicit. The Go Cloud v0.40.0
adapter uses this client for reads and uploads without choosing a checksum
policy. Backend incompatibility is a risk; no Prow deployment failure was
reproduced by this followup analysis.

Implement the policy already recorded in the review: add the aws import and
set these functional options before config.LoadDefaultConfig:
  config.WithRequestChecksumCalculation(aws.RequestChecksumCalculationWhenRequired)
  config.WithResponseChecksumValidation(aws.ResponseChecksumValidationWhenRequired)

Apply both settings unconditionally, including when Endpoint is empty.
WhenRequired limits automatic checksums; it does not disable checksums
required by an API or explicitly requested by an operation. Document why Prow
chooses this compatibility policy. Explicit code options override the
corresponding environment/shared-config settings and reduce optional integrity
checks for AWS S3 too.

Acceptance criteria:
- Client options contain both WhenRequired values for the default AWS endpoint
  and a custom endpoint.
- Add focused assertions to Test_newS3Client in pkg/io/providers/aws_test.go.
  Isolate AWS config and checksum environment variables so tests are stable.
- Existing credentials, region, endpoint, path-style, and TLS assertions pass.
- Run go test ./pkg/io/... and go build ./...; report their outcomes.

Scope: change only newS3Client's config construction and its focused tests.
Keep credential selection, endpoint handling, region, TLS, and path-style
behavior as they are. Do not upgrade other dependencies or claim a confirmed
MinIO/Ceph regression without reproducing one.
```
