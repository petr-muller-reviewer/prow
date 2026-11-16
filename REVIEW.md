---
pr: kubernetes-sigs/prow#1069
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: 8d4e2fe716ec5238674183802c80ca4d4f0310f9
base: main
reviewed_at: 2026-11-16T23:22:00Z
verdict: approve
---

# Review

## Verdict

Safe to bump now.

Dep-only PR: all four changed files are `go.mod`/`go.sum` for the root module and for `hack/tools`. The four direct bumps are ordinary AWS SDK patch releases (10, 10, 7 and 7 days old at review time), the indirect churn is the usual in-SDK module graph (`smithy-go`, `internal/v4a`, `service/sts`, `sso`, `ssooidc`, `signin`, `s3shared`, `checksum`, `presigned-url`, `accept-encoding`, `configsources`, `endpoints/v2`), and every added/removed `go.sum` line is an AWS module — no unrelated drift. OSV reports no advisories for any bumped module at either the old or the new version, `govulncheck` symbol scan over the only AWS-importing package is clean at base and head, the bump type-checks, and all Prow CI jobs pass. The one behavioural change in the range on a path we exercise (SigV4 canonical query parameters sorted after URI-encoding) is a spec-compliance fix that is a no-op for S3's plain-ASCII query keys.

## What this PR does

- Bumps `github.com/aws/aws-sdk-go-v2` v1.47.5 -> v1.47.8 and `github.com/aws/aws-sdk-go-v2/service/s3` v1.114.6 -> v1.114.9 (both from the 2026-11-06 SDK release train).
- Bumps `github.com/aws/aws-sdk-go-v2/config` v1.33.13 -> v1.33.17 and `github.com/aws/aws-sdk-go-v2/credentials` v1.20.13 -> v1.20.17 (2026-11-09 train).
- Pulls the matching indirect modules: `aws/smithy-go` v1.28.7 -> v1.28.8 plus `internal/v4a` v1.5.8 -> v1.5.11, `internal/configsources` v1.5.8 -> v1.5.11, `internal/endpoints/v2` v2.8.8 -> v2.8.11, `service/sts` v1.51.6 -> v1.51.9, `sso` v1.39.1 -> v1.39.4, `ssooidc` v1.44.1 -> v1.44.4, `signin` v1.12.1 -> v1.13.0, `s3shared` v1.20.8 -> v1.20.11, `checksum` v1.11.9 -> v1.11.12, `presigned-url` v1.14.8 -> v1.14.11, `accept-encoding` v1.13.23 -> v1.13.24, `eventstream` v1.7.25 -> v1.7.26, `feature/ec2/imds` v1.20.5 -> v1.20.8.
- Mirrors the same graph into `hack/tools/go.mod`, where all AWS modules are indirect (tooling chain: `awslabs/amazon-ecr-credential-helper`, ECR clients).

## Findings

None.

No blocking, should-fix or nit-level findings. The question below is about test coverage against non-AWS S3 endpoints, not a defect in this diff.

## Checked

- Classification: `git diff --stat` shows only `go.mod`, `go.sum`, `hack/tools/go.mod`, `hack/tools/go.sum` changed — dep-only, no project code, so no code review pass.
- Freshness (proxy.golang.org `.info`, `Origin.Ref` confirms real tags, not pseudo-versions): core v1.47.8 2026-11-06 (10 days), s3 v1.114.9 2026-11-06 (10 days), config/credentials v1.33.17/v1.20.17 2026-11-09 (7 days), smithy-go v1.28.8 2026-11-03 (13 days). Nothing inside the <5-day "too fresh" band; config/credentials at 7 days noted but the range is dependency-update-only for those two modules.
- Newer releases already exist as of 2026-11-16 (core v1.48.0, config v1.33.19, credentials v1.21.1, s3 v1.114.10, smithy-go v1.28.9) — this PR is a slightly stale dependabot snapshot, which is normal; the next run will supersede it.
- Vulnerabilities: OSV `v1` POSTs for `Go` ecosystem per module/version (core, config, credentials, service/s3, smithy-go x old+new) all returned `[]`. No advisory is fixed by the bump and none remains open.
- Reachability: `govulncheck` module scan is broken in this environment (v1.9.0, DB 2026-11-11: `-scan module` without patterns errors "no Go files in /workspace", with patterns errors "patterns are not accepted for module only scanning"). I substituted `govulncheck -format json ./pkg/io/...` (the only AWS-importing package) on both sides: zero findings at merge-base `86b2c49` and at head, and it type-checked successfully on both, which also confirms the bump builds. Given OSV is empty module-wide there is nothing a module-level scan could surface.
- CI: `pull-prow-unit-test`, `pull-prow-integration`, `pull-prow-image-build-test`, `pull-prow-verify-lint`, `pull-prow-unit-test-race-detector-nonblocking` all pass; `tide` is pending only on `approved`/`lgtm` labels.
- Usage: only `pkg/io/providers/aws.go` and `pkg/io/providers/aws_test.go` import AWS SDK v2 outside vendor. `go mod why -m` for `service/s3` and `config` both resolve through `sigs.k8s.io/prow/pkg/io/providers`. Direct deps: `aws-sdk-go-v2`, `config`, `credentials`, `service/s3`; `smithy-go` and the rest are `// indirect`.
- Changelog review, core v1.47.6..v1.47.8 / s3 v1.114.7..v1.114.9 / config v1.33.14..v1.33.17 / credentials v1.20.14..v1.20.17: rolled-up "dependency update" releases plus two real fixes — (a) clock skew now computed from the server `Date` header minus response-receive time instead of the request/response midpoint (core v1.47.7; exercised on every S3 call, an improvement for long uploads/downloads), (b) SigV4 canonical query parameters sorted after URI-encoding (`aws/signer/v4` -> `v4Internal.EncodeQuery`, via `internal/v4a` v1.5.11 / `aws-http-auth` v1.2.2; the v1.47.5->v1.47.8 diff of `aws/signer/v4/v4.go` confirms the change is in this bump). No credential-chain or auth logic changes in `config`/`credentials` in this range — v1.20.12's AssumeRole-missing-credentials panic fix is already in our old version.
- smithy-go v1.28.8 (`Fix malformed JSON when serializing nil lists or maps...`, event-stream fixes, JSON number precision): S3 uses REST-XML, not JSON/awsQuery schema serde, so not on our path.
- Future-bump watch item (not this PR): `credentials` v1.21.0 (2026-11-10) makes Retry Behavior 2.1 the default — new backoff base (50ms, 1s for throttling), `x-amz-retry-after` honouring, and 429s from IMDS/endpoint credentials retried as throttling. That will change retry behaviour for our S3 calls and deserves a closer look when dependabot picks it up; s3 v1.114.10 is an S3 Express credentials fix we don't use.

## Open questions

- Prow supports non-AWS S3 by setting `BaseEndpoint`/`S3ForcePathStyle` (`pkg/io/providers/aws.go:100-112`), e.g. MinIO or Ceph RGW. This bump changes SigV4 canonical query ordering (sort after URI-encoding, per spec). For S3's plain-ASCII query keys (`list-type`, `prefix`, `continuation-token`, `max-keys`) the ordering is unchanged, and the integration test jobs only exercise fakes — was this bump smoke-tested against a non-AWS S3 endpoint, or do we rely on those implementations being spec-compliant here? No action expected; asking for the record.

```
pkg/io/providers/aws.go:100
	if creds.Endpoint != "" {
		s3Opts = append(s3Opts, func(o *s3.Options) {
			o.BaseEndpoint = &creds.Endpoint
		})
	}
```

## Dependency followups

No opportunities identified. Walked all modules bumped by PR #1069 (open, head `8d4e2fe`), old -> new, including intermediates:

- Direct: `aws-sdk-go-v2` v1.47.5 -> v1.47.8, `aws-sdk-go-v2/config` v1.33.13 -> v1.33.17, `aws-sdk-go-v2/credentials` v1.20.13 -> v1.20.17, `aws-sdk-go-v2/service/s3` v1.114.6 -> v1.114.9.
- Transitive: `aws/smithy-go` v1.28.7 -> v1.28.8, `aws/protocol/eventstream` v1.7.25 -> v1.7.26, `feature/ec2/imds` v1.20.5 -> v1.20.8, `internal/configsources` v1.5.8 -> v1.5.11, `internal/endpoints/v2` v2.8.8 -> v2.8.11, `internal/v4a` v1.5.8 -> v1.5.11, `service/internal/accept-encoding` v1.13.23 -> v1.13.24, `service/internal/checksum` v1.11.9 -> v1.11.12, `service/internal/presigned-url` v1.14.8 -> v1.14.11, `service/internal/s3shared` v1.20.8 -> v1.20.11, `service/signin` v1.12.1 -> v1.13.0, `service/sso` v1.39.1 -> v1.39.4, `service/ssooidc` v1.44.1 -> v1.44.4, `service/sts` v1.51.6 -> v1.51.9.

Reason: every changelog entry in these ranges is a "dependency update" roll-up, plus internal bug fixes we do not call (clock-skew-from-receive-time, SigV4 canonical query ordering, smithy schema-serde JSON/nil-collection fixes) and one new API in a module we never import (`signin` v1.13.0 `CreateBearerToken`). No deprecations in range touch `pkg/io/providers/aws.go`, which is the only AWS-importing package — it uses `config.LoadDefaultConfig`/`WithCredentialsProvider`/`WithRegion`, `credentials.NewStaticCredentialsProvider`, `s3.NewFromConfig`/`s3.Options`, `awshttp.NewBuildableClient().WithTransportOptions` and `aws.RequestChecksumCalculation*`, none of which changed or were deprecated.
