---
issue: kubernetes-sigs/prow#970
title: "Pod-utility uploads to S3-compatible stores fail with `501 NotImplemented: AWS chunked encoding not supported`"
state: open
labels: []
main_sha: a490515e5655cd08fd602e3bbef6a6f3cbfea16b
triaged_at: 2026-09-27T14:03:26Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/bug, area/provider/aws, area/podutils/initupload, help-wanted]
---

## Initial validation

**Assessment**: LEGITIMATE

### Analysis

This is a specific, reproducible storage-provider compatibility regression in Prow-owned code. The reporter supplies the failing operation and response (`PutObject`, HTTP 501), the required S3-compatible configuration, the SDK behavior that introduces `aws-chunked`, and a scoped fix. The deployment has reached the object store, so the report is not a credentials or endpoint-configuration support request.

The issue is already being addressed by open PR #971, from the reporter, which changes the shared S3-client construction path and adds focused unit coverage. No duplicate issue was found: searches for `aws chunked`, `s3 upload`, and `pod utility` returned #970 as the only matching report. The issue timeline cross-references PR #971.

**Issue Category**: Bug

**Repository Scope Check**:
- Component mentioned: pod-utility artifact upload through Prow's S3 provider.
- Exists in this repo: Yes.
- Relevant code paths: `pkg/pod-utils/gcs/upload.go:55-72`, `pkg/io/opener.go:113-130`, `pkg/io/providers/providers.go:75-82`, and `pkg/io/providers/aws.go:38-92`.

**Information Completeness**:
- Sufficient detail provided: Yes.
- Missing information: No blocker. An OCI-backed end-to-end reproduction would strengthen the regression test but is not needed to establish the client-configuration defect.

### Recommendation

Keep open while PR #971 is reviewed. It is a well-scoped bug in the repository's explicit S3-compatible endpoint support, and the existing PR implements the proposed low-risk behavior.

**Suggested Action**:
- Keep open and continue triage; review PR #971.

## Research findings

### How it works today

Pod-utility upload constructs an `io.Opener` with the mounted S3 credentials file (`pkg/pod-utils/gcs/upload.go:55-72`; `pkg/pod-utils/decorate/podspec.go:611-624`). The opener reads that file once and, for an `s3://` URL, routes it to `providers.GetBucket` (`pkg/io/opener.go:113-130`, `pkg/io/opener.go:207-227`). `GetBucket` chooses the explicit S3 path when credentials are supplied, then `getS3Bucket` unmarshals `S3Credentials`, constructs an AWS SDK v2 client, and passes it to Go CDK's `s3blob.OpenBucketV2` (`pkg/io/providers/providers.go:75-101`, `pkg/io/providers/aws.go:38-51`).

`newS3Client` loads the SDK default configuration and applies Prow's static credentials, region, HTTP/TLS, path-style, and optional `BaseEndpoint` settings (`pkg/io/providers/aws.go:54-92`). It does not set request-checksum policy. The checked-in AWS SDK documents that its default is `RequestChecksumCalculationWhenSupported`, which calculates checksums whenever the operation supports them; `WhenRequired` calculates only when required or selected by the caller.

### Root cause

For any explicitly configured custom endpoint, Prow uses the AWS SDK's default request-checksum policy. On streamed S3 `PutObject` operations, that policy can result in a trailing checksum encoded as `Content-Encoding: aws-chunked`; OCI Object Storage and other partial S3 implementations reject it with the observed 501. Confidence is high: the failure occurs after the request reaches `PutObject`, Prow's custom-endpoint path is present, and the SDK's current documented default matches the reporter's account.

This is deployment-visible after AWS SDK v2 upgrades, but the exact upstream SDK release that first made this request shape observable has not been isolated; the large November 2025 dependency upgrade moved the core SDK from v1.32.4 to v1.36.3. That provenance is not needed for the proposed fix.

### Tests and docs

Partial coverage. `pkg/io/providers/aws_test.go:29-119` verifies client credentials, endpoint, region, path style, and TLS settings, but not checksum policy. `pkg/io/providers/providers.go:58-74` documents custom S3 endpoints, which establishes compatibility as an intended use case. There is no OCI/Swift integration test, so the wire-level `aws-chunked` rejection is currently untested.

### Proposed approach

**Recommended**: Default custom endpoints to `RequestChecksumCalculationWhenRequired` — after `config.LoadDefaultConfig`, set that policy only when `S3Credentials.Endpoint` is non-empty and `AWS_REQUEST_CHECKSUM_CALCULATION` is unset. This is the implementation in PR #971. It preserves SDK-default checksum behavior for AWS S3 and preserves an operator's environment override, while making the documented custom-endpoint path interoperable with stores that do not implement AWS chunked encoding.

- Trade-offs: custom endpoints no longer get an opportunistic SDK-generated request checksum unless the operation requires one or the operator explicitly requests it. This favors broad S3-compatible interoperability over an optional integrity feature that those endpoints cannot process.
- Backwards compatibility: no change for Prow users without `endpoint`; custom endpoint users can retain `when_supported` through the existing SDK environment variable.
- Testing needed: table-driven unit tests for no endpoint, endpoint with no override, and endpoint with `AWS_REQUEST_CHECKSUM_CALCULATION=when_supported`; ideally an integration test against an endpoint that rejects `aws-chunked` when practical.

**Alternative**: expose an environment-variable list through `DecorationConfig` and inject `AWS_REQUEST_CHECKSUM_CALCULATION` into pod-utility containers. This is more general, but expands public configuration, deepcopy/validation, and container wiring for a policy Prow can select safely based on its already-explicit custom endpoint.

## Effort assessment

**Effort Level**: 2 - Moderate

The selected change is small and clear, but it changes SDK request semantics for every configured S3-compatible endpoint and needs careful preservation of the SDK's environment/profile precedence.

- **Solution certainty**: clear — PR #971 implements the narrow endpoint-only policy and tests the intended precedence.
- **Blast radius**: multiple components — shared S3 client construction affects initupload, sidecar/gcsupload, and any Prow caller that uses `io.Opener` with S3 credentials.
- **Expertise**: AWS SDK v2 checksum settings, Prow's storage-provider path, and S3-compatible endpoint behavior.
- **Scope estimate**: Small — `pkg/io/providers/aws.go` plus focused provider tests; optional compatibility integration coverage.

**Recommended labels**: `kind/bug` (Prow's supported custom S3 endpoint path fails), `area/provider/aws` (AWS SDK v2 client policy), `area/podutils/initupload` (reported user-facing failure), `help-wanted` (well-defined implementation already exists in PR #971).

**Guidance for contributors**: Start with `newS3Client` and its tests. Retain the existing SDK environment override, keep the behavior conditional on `Endpoint`, and avoid broadening the ProwJob API merely to set an SDK policy.

## Briefing summary

Issue #970 is a legitimate compatibility bug in Prow's shared AWS S3 client: its default opportunistic request checksum can make streamed uploads use `aws-chunked`, which configured S3-compatible endpoints such as OCI reject. The narrowly scoped, backwards-compatible fix in open PR #971 defaults only custom endpoints to `when_required` while preserving the existing SDK environment override. The work is Level 2 because the implementation is small but affects all configured compatible endpoints.

Briefed maintainer on: 2026-09-27T14:03:26Z

Key questions asked:
- None.

Maintainer decision:
Not yet indicated.

## Findings

- [reproducibility] A ProwJob configured with an explicit S3-compatible `endpoint` reaches `PutObject`, but OCI Object Storage returns `501 NotImplemented` for `aws-chunked` request encoding.
- [cause] `pkg/io/providers/aws.go:54-92` leaves the SDK on its default `RequestChecksumCalculationWhenSupported`; the SDK documents that this calculates request checksums whenever supported, which triggers incompatible streaming behavior for the reported stores.
- [related-code] `pkg/pod-utils/gcs/upload.go:55-72` builds the opener used by artifact upload; `pkg/io/opener.go:207-227` caches buckets and delegates S3 construction to the provider.
- [related-code] `pkg/io/providers/providers.go:75-101` explicitly supports S3-compatible endpoints and routes supplied S3 credentials to `getS3Bucket`.
- [related-pr] #971 (open) implements endpoint-only `when_required` defaulting with tests for AWS S3, custom endpoints, and the environment override.

## Checked

- Queried issue searches: `aws chunked`, `s3 upload`, and `pod utility`; no duplicate issue was found.
- Checked #970's timeline; it cross-references open PR #971.
- Read the shared upload, opener, provider, and AWS client paths, their unit tests, and the SDK's current request-checksum contract.
- Checked PR #971 discussion/status; it is mergeable, has no human reviews yet, and no failing completed check was reported.
- Reviewed history of the AWS client path; the custom-endpoint support predates the dependency update. The precise upstream version that altered observed wire behavior remains unproven.

## Next steps

- Review PR #971, paying particular attention to environment/profile precedence and the custom-endpoint-only guard.
- Apply `kind/bug`, `area/provider/aws`, `area/podutils/initupload`, and `help-wanted` if the maintainer agrees.
- Keep #970 open until #971 merges; then close it as fixed.

## Open questions

- Should an OCI/Swift-compatible integration fixture be added, or are the targeted SDK-client policy tests sufficient for this dependency behavior?
- Is `area/podutils/initupload` preferred over the broader `area/pod-utilities`, given that the same shared path can also affect other pod utilities?
