---
pr: kubernetes-sigs/prow#826
title: "chore(deps): bump the aws group across 1 directory with 4 updates"
head_sha: f3371bdd561f5c793fe165cf823f30ec26c0a559
base: main
reviewed_at: 2026-08-11T15:27:42Z
verdict: approve
---

## Summary

Dependabot group bump of AWS SDK v2 modules. Dep-only PR: only `go.mod`/`go.sum` (root and `hack/tools`) changed, no project code.

Direct deps bumped (same versions in both go.mod files):
- `github.com/aws/aws-sdk-go-v2` v1.43.0 -> v1.43.4
- `github.com/aws/aws-sdk-go-v2/config` v1.32.31 -> v1.32.35
- `github.com/aws/aws-sdk-go-v2/credentials` v1.19.30 -> v1.19.34
- `github.com/aws/aws-sdk-go-v2/service/s3` v1.106.0 -> v1.107.0

Plus indirect submodules riding the same AWS SDK v2 release train (`aws/protocol/eventstream`, `feature/ec2/imds`, `internal/configsources`, `internal/endpoints/v2`, `internal/v4a`, `service/internal/*`, `service/signin`, `service/sso`, `service/ssooidc`, `service/sts`) and `github.com/aws/smithy-go` v1.27.3 -> v1.27.6.

## Findings

### [question] Release freshness right at the soak-time edge
- where: `go.mod:14-17`
- concern: New tags were cut 2026-08-05 (aws-sdk-go-v2 core/config/credentials, commit `ebd1f6a`) and 2026-08-06 (service/s3) — 5-6 days old as of review (2026-08-11). Not flagged as dangerously fresh, but close to the usual wait-for-soak threshold. No CVE or urgent fix in the range that would justify skipping the wait.
- excerpt: |
    github.com/aws/aws-sdk-go-v2 v1.43.4
    github.com/aws/aws-sdk-go-v2/config v1.32.35
    github.com/aws/aws-sdk-go-v2/credentials v1.19.34
    github.com/aws/aws-sdk-go-v2/service/s3 v1.107.0

## Checked
- Classified as dep-only: diff touches only `go.mod`/`go.sum` (root and `hack/tools`), no project source changed.
- Import surface for all four direct deps: only `pkg/io/providers/aws.go` (S3 blob storage provider) — credential loading, custom TLS transport, S3 client construction. Sensitive but narrow (single file).
- Changelog for the v1.43.0..v1.43.4 range (aws-sdk-go-v2/config/credentials share the same tag commit): no CVEs; mostly kitchen-sink test infra, codegen/model regen, smithy-go bumps. One relevant fix: "Expand checks for S3 200 errors to more operations (wave 1)" (#3493) — improves detection of S3's HTTP-200-with-embedded-error quirk, directly applicable to our `s3.NewFromConfig` usage via `s3blob`, low regression risk.
- "Set FIPS-approved TLS curve preferences when FIPS module is active" (#3499) — not exercised, `aws.go` doesn't opt into FIPS mode.
- Transfer Manager `HeadObject` field additions (#3491) — not applicable, we don't use Transfer Manager.
- No pseudo-versions; all tags resolve cleanly via the Go module proxy with proper `Origin` metadata.
- Investigated `pull-prow-integration` CI failure (run 2087143800412573696): unrelated to this PR. `test/integration/integration-test.sh` installs Tekton Pipeline v1.6.0 into a KIND cluster; the `tekton-pipelines-webhook` pod stuck in `ImagePullBackOff` because containerd/kubelet repeatedly hit `net/http: TLS handshake timeout` pulling `ghcr.io/tektoncd/pipeline/webhook-...:v1.6.0` from `pkg-containers.githubusercontent.com` (8 failures over ~7 min, same error for the `resolvers` image too). No code path connects the AWS SDK v2 bump to Tekton image pulls — this is a registry/network flake in CI infra, not caused by the diff. Recommend `/retest`.

## Open questions
- Given the releases are only 5-6 days old, is there urgency to merge now, or can this wait a few more days for additional soak time?

## Dependency followups

None. Checked changelogs for all in-scope modules (aws-sdk-go-v2 core, config, credentials, service/s3) between old and new versions against our only usage site (`pkg/io/providers/aws.go`):

- `config`/`credentials`: range contains only "Dependency Update" entries, no API changes.
- aws-sdk-go-v2 core / `smithy-go`: range is kitchen-sink test infra, codegen/model regen, and serde fixes for HTTP binding services — no public API touched by our code. `smithy-go` isn't imported directly by us.
- `service/s3`: two substantive changes — a bug fix (#3493, "Expand S3 operations that check for an error inside an HTTP 200 response") that's internal SDK behavior with no call-site change required, and a new feature (S3 Backup read-only access points) unrelated to our blob-storage usage.
- No deprecated APIs in the changelog match anything called in `newS3Client`/`getS3Bucket` (`config.LoadDefaultConfig`, `config.WithCredentialsProvider`, `config.WithRegion`, `credentials.NewStaticCredentialsProvider`, `s3.NewFromConfig`, `awshttp.NewBuildableClient`).

Tally: 0 accepted, 0 skipped.
