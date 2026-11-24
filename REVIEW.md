---
pr: kubernetes-sigs/prow#1088
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: 194901aae1c24f5c113493c1ea50c446272972a7
base: main
reviewed_at: 2026-11-24T13:08:25Z
verdict: approve
---

# Review

## Verdict
Safe to bump now — dep-only grouped patch bump of AWS SDK v2, no advisories on either side, one behavior change (retry policy) worth a heads-up rather than a hold.

The four direct modules are the S3 artifact-storage backend (`pkg/io/providers/aws.go`), which is on a core path (artifact upload/download for deck, plank, `pod-utils/gcs`, spyglass, resultstore, crier) but is exercised by exactly one provider package. The only change that lands in code we run is Retry Behavior 2.1 becoming the SDK default in v1.48.0; the rest is generated-client/model churn, an S3 Express fix we don't exercise, and an SSO token-refresh fix. Freshness is acceptable: 14–15 days for the core/s3/smithy releases, 8 days for `config`/`credentials`, and the group lags the newer upstream patches (config 1.33.21, credentials 1.21.3, s3 1.116.0), so it is not a bleeding-edge pick. `go mod tidy -diff` is clean for both modules, `go build`/`go test ./pkg/io/...` pass, and `govulncheck` at base vs head shows no new or fixed findings in any AWS module.

## What this PR does
- Bumps the four direct AWS SDK v2 modules in the root module: `aws-sdk-go-v2` 1.47.8→1.48.0, `config` 1.33.17→1.33.19, `credentials` 1.20.17→1.21.1, `service/s3` 1.114.9→1.114.10.
- Carries the matching fan-out of indirect AWS modules (imds, configsources, endpoints/v2, v4a, accept-encoding, checksum, presigned-url, s3shared, signin, sso, ssooidc, sts, `smithy-go` 1.28.8→1.28.9) in `go.mod`/`go.sum`.
- Applies the same version set to `hack/tools/go.mod`, where all of them are indirect (ko → awslabs ecr-login → config → sts), i.e. build-tooling only.
- No project source, tests, config or docs are touched; no `vendor/` tree in this repo.

## Findings

None. No blocking, should-fix, or nit issues.

## Checked
- PR classification: dep-only — `git diff --stat 901165669..HEAD` is exactly `go.mod`, `go.sum`, `hack/tools/go.mod`, `hack/tools/go.sum`.
- Freshness from the module proxy: `aws-sdk-go-v2` v1.48.0 and `service/s3` v1.114.10 are 2026-11-10 (14 days), `smithy-go` v1.28.9 is 2026-11-09 (15 days), `config` v1.33.19 and `credentials` v1.21.1 are 2026-11-16 (8 days). All real tags, no pseudo-versions.
- Not the freshest upstream: config v1.33.21, credentials v1.21.3, s3 v1.116.0 already exist (11-17…11-19), so the picked versions have a further patch round of soak.
- OSV (`api.osv.dev`) for old and new versions of all four direct modules plus `smithy-go`, `sts`, `sso`, `ssooidc`, `signin`, `feature/ec2/imds`: zero advisories on both sides. Nothing fixed, nothing left open.
- `govulncheck` (v1.10.0, DB 2026-11-23): `-scan module` cannot run here (fails with `no Go files in /workspace` — no package at the module root, and patterns are rejected in module mode), so a package-level scan was used instead. Base `901165669` and PR head produce identical findings — 17 entries, all `GO-2023-1901` (tektoncd/pipeline) and `GO-2026-5932` (golang.org/x/crypto); none in any AWS module, so no new reachable vulns and none fixed. The head scan also type-checks the whole tree, so the bump builds.
- Usage: `pkg/io/providers/aws.go` (plus `aws_test.go`) is the only importer — `config.LoadDefaultConfig`, optional `credentials.NewStaticCredentialsProvider`, `s3.NewFromConfig` with path-style/custom endpoint and optional insecure TLS, wrapped by `gocloud.dev/blob/s3blob` (`aws.go:60-112`). Consumed through `pkg/io` by deck, plank, `pod-utils/gcs` upload, spyglass, resultstore and crier. Sensitive domain (credentials, TLS, network, object I/O), light code footprint, heavy path when S3 is the configured backend.
- `aws-sdk-go-v2` v1.48.0 changelog: Retry Behavior 2.1 is now the default (`AWS_NEW_RETRIES_2026` removed) — unmodeled HTTP 429s, including from IMDS and the endpoint credentials provider, retry as throttling; 50 ms base backoff (1 s for throttling); `x-amz-retry-after` honored; `StandardOptions.RetryTimeoutCost`/`Timeouts` deprecated and unused. We use the default retryer with no overrides, so max attempts stay at 3 and only timing/429 handling shifts. This is the one behavior change we exercise.
- `service/s3` v1.114.10: only the S3 Express `CreateSession` signing/cache-key fix; we do not use S3 Express (explicit endpoints, path style), so no impact.
- `config` v1.33.19: dependency updates only. `credentials` v1.21.1: SSO login-token refresh uses the token's `signInRegion` — SSO profiles only, not the in-cluster IMDS/static-credential path.
- `smithy-go` v1.28.9: SigV4 canonical query params sorted after URI-encoding (aws-http-auth v1.2.2), schema-serde fixes for awsQuery/XML, and a nil-pointer marshal panic fix. S3 is restxml and we sign path-style requests — correctness fix, negligible risk.
- `hack/tools`: all AWS modules indirect via ko → ecr-login; no runtime effect.
- Verification: `go mod tidy -diff` clean for `go.mod` and `hack/tools/go.mod`; `go build ./pkg/io/...` and `go test ./pkg/io/...` pass; `go.mod`/`go.sum` diff contains only the expected AWS versions.

## Open questions
- Retry Behavior 2.1 lands in every AWS call in this bump, including S3 uploads against non-AWS S3 gateways (the OCI/Swift setups that already need `request_checksum_calculation: when_required`). Do we have any S3 e2e that exercises throttling/429 against such a gateway, or is the plan to rely on the two weeks of upstream soak? If not, that is the one thing I would watch after merge.
- The group is two patch rounds behind upstream (`config` 1.33.21, `credentials` 1.21.3, `s3` 1.116.0). Fine to merge as-is and let the next dependabot run pick those up — flagging only so nobody assumes this is the latest.

## Dependency followups

No improvement opportunities identified (0 accepted, 0 skipped). Examined the full changelog range of every bumped module and cross-referenced it against our only importer, `pkg/io/providers/aws.go` (plus `aws_test.go`):

- `aws-sdk-go-v2` v1.47.8→v1.48.0 — Retry Behavior 2.1 default is automatic (we set no retry options, so nothing to drop or adopt); `StandardOptions.RetryTimeoutCost`/`Timeouts` are the only new deprecations and we never construct retry options; S3 Express signing, schema-serde awsquery and nil-pointer marshal fixes touch APIs we don't use.
- `service/s3` v1.114.9→v1.114.10 — S3 Express `CreateSession` fix only; we use plain path-style S3 via `s3blob`.
- `config` v1.33.17→v1.33.19 — dependency-update entries only.
- `credentials` v1.20.17→v1.21.1 — retry-default rollover plus an SSO `signInRegion` token-refresh fix; we use static credentials / the default chain, not SSO profiles.
- `smithy-go` v1.28.8→v1.28.9 — SigV4 canonical query-parameter ordering fix (automatic), awsQuery/XML schema-serde fixes (S3 is restxml), and a marshal panic fix for pointer-receiver `Marshaler` types we don't define.
- Transitive-only AWS modules (eventstream, imds, configsources, endpoints/v2, v4a, accept-encoding, checksum, presigned-url, s3shared, signin, sso, ssooidc, sts) and `hack/tools` — patch-level dependency-update churn with nothing we call directly.

Grepped our tree for every deprecated symbol in the new versions (`EndpointResolver`, `RetryMode`/`RetryMaxAttempts` overrides, `StandardOptions.*`, `aws.Config` literals, `aws/retry` imports): no matches outside `vendor/`.
