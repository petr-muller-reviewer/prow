---
issue: kubernetes-sigs/prow#970
title: "Pod-utility uploads to S3-compatible stores fail with `501 NotImplemented: AWS chunked encoding not supported`"
state: closed
labels: kind/bug, area/podutils/initupload
main_sha: a490515e5655cd08fd602e3bbef6a6f3cbfea16b
triaged_at: 2026-10-29T14:27:22Z
verdict: resolved
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/bug, area/provider/aws, area/podutils/initupload, help-wanted]
refresh_log:
  - at: 2026-10-29T14:27:22Z
    since: 2026-09-27T14:03:26Z
    summary: "Issue closed COMPLETED by merged PR #971; resolution check found the shipped opt-in credential field sufficient; recorded the gocloud transfer-manager correction and the label changes."
---

## Resolution

Resolving PR: kubernetes-sigs/prow#971 (merged 2026-10-28T08:53:37Z, squash commit `707da3e68`, merge commit `c06373a3`), authored by the reporter, Sunnatillo. The issue was closed `COMPLETED` one second later, and the squash commit is in `upstream/main`.

**Classification**: sufficient.

- Fixes the triaged cause: `pkg/io/providers/aws.go:81-93` maps a new optional `S3Credentials.RequestChecksumCalculation` field (`when_required` / `when_supported`, case-insensitive, invalid values rejected) onto the SDK config, and `pkg/io/providers/aws.go:49-52` forwards it to `s3blob.Options` so gocloud's transfer manager honors it. `gocloud.dev` is bumped to v0.47.0 (`go.mod:63`), and `pkg/io/providers/providers.go:75-82` documents the OCI/Swift recipe.
- Wire-level evidence exists at both ends: `Test_getS3Bucket_UploadChecksumHeaders` in `pkg/io/providers/aws_test.go` drives a real write through a TLS test server and asserts presence/absence of `Content-Encoding: aws-chunked` and `X-Amz-Trailer`; the author's OCI `eu-paris-1` verification on PR #971 reports 501 for the default and 200 with `when_required` (`X-Amz-Content-Sha256: UNSIGNED-PAYLOAD`, no trailer).
- Deviation from the recommendation, not from the cause: the shipped change is an opt-in per-credential-file setting rather than automatic `when_required` for every non-empty `endpoint`. Default behavior is unchanged, so an affected deployment still needs `{"request_checksum_calculation": "when_required"}` after upgrading. The issue body now documents this design, and maintainers (lentzi90 LGTM, petr-muller `/approve`) accepted it without asking for automatic defaulting; no post-merge comment reports the failure persisting.
- The previously recommended next step (review #971 for environment precedence and the custom-endpoint guard) was followed: the merged tests cover unset/invalid values, field-over-env precedence, and the environment fallback, but the custom-endpoint guard was dropped along with automatic defaulting.

## What the issue reports

- Every ProwJob stops at init (`Init:Error`): `initupload` cannot write to OCI Object Storage and fails with `501 NotImplemented: AWS chunked encoding not supported`.
- The AWS SDK v2 default checksum mode (`when_supported`) makes a streamed `PutObject` send `Content-Encoding: aws-chunked` with a CRC32 trailer; AWS S3, MinIO, and recent Ceph RGW accept it, while OCI Object Storage and OpenStack Swift reject it.
- Prow's S3 credential path had no way to select `when_required`, and pod-utility containers do not get an environment-variable escape hatch, so affected deployments had no workaround.
- Requested: a per-credential-file `request_checksum_calculation` setting (`when_supported` / `when_required`; empty keeps the SDK default) that also reaches the gocloud upload path.

Since previous triage (2026-09-27T14:03:26Z):
- Closed as `COMPLETED` on 2026-10-28T08:53:38Z by merged PR #971; labels changed from none to `kind/bug` and `area/podutils/initupload`.
- The issue body now documents the final opt-in design (optional `request_checksum_calculation` field plus a `gocloud.dev` bump), which differs from the endpoint-only defaulting this triage originally described.
- New issue comments are process-only: `/kind bug` and `/area podutils/initupload` (petr-muller, 2026-09-27T14:06:18Z), a review request to the reporter and that reporter accepting (petr-muller 14:19:14Z, Ultronious 14:24:49Z). The OCI verification evidence lives in PR #971, not the issue.

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

**Shipped (PR #971, merged 2026-10-28)**: neither option above. The merged change keeps the SDK default and adds an optional `request_checksum_calculation` field to `S3Credentials` (`pkg/io/providers/aws.go:81-93`), then forwards the resulting policy to `s3blob.Options` (`pkg/io/providers/aws.go:49-52`) with `gocloud.dev` v0.47.0. The reporter chose this because the credential file already reaches `initupload` and `sidecar`, whereas an environment variable does not reach them. It is strictly smaller than the alternative and does not change behavior for existing users, but it also does not remove the 501 for a deployment that upgrades without changing its secret.

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
- [cause] Setting the policy on `s3.Client` alone is insufficient: gocloud's S3 writer uploads through the SDK transfer manager, whose own checksum setting overrides the client's. The merged fix therefore passes `client.Options().RequestChecksumCalculation` to `s3blob.Options.RequestChecksumCalculation` (`pkg/io/providers/aws.go:49-52`), which required `gocloud.dev` v0.47.0.
- [related-pr] #971 (merged 2026-10-28, squash commit `707da3e68`) adds the opt-in `request_checksum_calculation` credential field with table-driven client tests, an invalid-value error, field-over-env precedence, and a TLS-server test asserting the real `PutObject` headers.

## Resolved

- [related-pr] #971 implemented endpoint-only `when_required` defaulting with tests for AWS S3, custom endpoints, and the environment override — superseded: the merged design is an opt-in credential field with unchanged defaults, plus a `gocloud.dev` bump.
- The triage's implied assumption that configuring `s3.Client` would change upload behavior — answered by the PR: the transfer manager has its own checksum setting; see the corrected `[cause]` finding above.
- Requested labels `area/provider/aws` and `help-wanted` — not applied; maintainers added only `kind/bug` and `area/podutils/initupload`.

## Checked

- Queried issue searches: `aws chunked`, `s3 upload`, and `pod utility`; no duplicate issue was found.
- Checked #970's timeline; it cross-references open PR #971.
- Read the shared upload, opener, provider, and AWS client paths, their unit tests, and the SDK's current request-checksum contract.
- Checked PR #971 discussion/status; it is mergeable, has no human reviews yet, and no failing completed check was reported.
- Reviewed history of the AWS client path; the custom-endpoint support predates the dependency update. The precise upstream version that altered observed wire behavior remains unproven.
- Re-checked #970 on 2026-10-29: state `CLOSED` / `COMPLETED`, closed 2026-10-28T08:53:38Z, closing reference PR #971, labels `kind/bug` and `area/podutils/initupload`.
- Read PR #971's final diff, reviews (lentzi90 LGTM 2026-10-23T07:54:43Z; petr-muller `/approve` twice, 2026-10-27T18:33:06Z and 2026-10-28T08:52:16Z), and its OCI verification comment (Sunnatillo, 2026-10-19T20:35:24Z).
- Confirmed the fix is on `upstream/main` (squash commit `707da3e68`, ancestor of `upstream/main`; 136 commits ahead of the triage's `main_sha`), and `upstream/main:go.mod` pins `gocloud.dev v0.47.0`.
- Checked for post-merge issue comments and cross-references: none; the only post-triage timeline events are the two label events and the close.

## Next steps

- No blocking maintainer action: #971 is merged and #970 is closed `COMPLETED`.
- Optional follow-ups if desired:
  - Document `request_checksum_calculation: "when_required"` in user-facing storage documentation, not only the `GetBucket` doc comment (`pkg/io/providers/providers.go:75-82`); upgrading alone does not fix an OCI/Swift deployment.
  - Consider a follow-up issue for automatic `when_required` defaulting on custom endpoints (this triage's original recommendation) if other adopters hit the 501 without reading the credential docs.

## Open questions

- Should checksum mode default to `when_required` for custom endpoints instead of requiring a credential-file opt-in? The shipped fix leaves defaults unchanged, trading interoperability for opportunistic integrity checks.
- Is a real OCI/Swift integration fixture worth adding beyond the TLS-server header assertion in `pkg/io/providers/aws_test.go`?
