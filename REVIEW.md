---
pr: kubernetes-sigs/prow#971
title: "Avoid aws-chunked 501 error on S3-compatible stores"
head_sha: b7d12482d40c24417b3536f4c97f107e4837f5ac
base: main
reviewed_at: 2026-10-23T13:45:34Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-27T18:31:34Z
  gated_head_sha: b7d12482d40c24417b3536f4c97f107e4837f5ac
  reviewed_head_sha: b7d12482d40c24417b3536f4c97f107e4837f5ac
---

# Review

## Gate

**Decision: merge.** PR #971 remains open at `b7d1248`, the same head covered by the saved review. No changes have landed since that review. The reported upload-path issue is addressed, and the only substantive inline request in the PR was for an explicit `when_supported` test, which the author added; the reviewer then left an LGTM.

### Findings disposition

- No unresolved gating findings. The request for a `when_supported` case in `pkg/io/providers/aws_test.go` was addressed in the current head. The author also reported testing both checksum modes against OCI Object Storage, with `when_required` returning 200 and the default returning 501.

### Independent merge risk

- `pkg/io/providers/providers.go:113` adds an exported field to `S3Credentials`. Downstream Go source using unkeyed composite literals may need a compile-time update; keyed literals and deployed JSON credential files are unaffected. No unkeyed literals occur in this repository, and this does not require a coordinated deployment rollout.
- The new JSON setting is opt-in, so existing credentials retain the SDK/environment checksum mode. The `gocloud.dev` and transfer-manager upgrades carry the setting into uploads; the request-level test covers the resulting headers. No other notable API, configuration, or behavioral risk was found.

### Gating list

- None.

## Verdict

Approve: no actionable findings.

The PR passes the configured checksum mode from the AWS client into Go Cloud's S3 transfer manager, addressing the upload-path gap from the earlier review. The new TLS-backed test checks actual upload headers for the default and `when_required` modes. I inspected the implementation and tests but did not execute the test suite.

## What this PR does

- Adds `request_checksum_calculation` to the S3 credentials JSON, supporting `when_supported` and `when_required`.
- Preserves AWS SDK configuration and environment behavior when the new field is empty, and rejects unsupported values.
- Passes the resolved checksum mode into Go Cloud's S3 bucket options so uploads use it.
- Adds tests for configuration precedence and request headers, and documents the setting for compatible stores.

## Findings

No findings.

## Checked

- Traced `RequestChecksumCalculation` from credentials and AWS SDK configuration through `getS3Bucket` into Go Cloud's transfer manager.
- Inspected Go Cloud v0.47.0's S3 writer behavior and the new TLS-backed test that captures `PutObject` headers.
- Reviewed the added tests for explicit values, environment precedence, invalid values, and actual upload headers. Tests were not run.

## Open questions

None.

## Followups

### [tests] Probe multipart upload checksum behavior

- category: tests
- necessity: could
- mode: probe
- where: `pkg/io/providers/aws_test.go`; `pkg/io/providers/aws.go:51`

**Why followup:** The merged regression test writes five bytes, exercising only a single `PutObject`. Prow artifacts can exceed the transfer manager's 16 MiB multipart threshold, so the large-upload path is not covered in Prow even though the fix exists to support stores that reject `aws-chunked`.

**Handoff prompt:**

```text
In `kubernetes-sigs/prow`, following merged PR #971 ("Avoid aws-chunked 501 error on S3-compatible stores", merge commit `c06373a3e2cdcf0de2a087b4ed1db426366694b8`), probe the multipart upload path in `pkg/io/providers`.

Background: `getS3Bucket` passes `client.Options().RequestChecksumCalculation` into `s3blob.Options`. The existing `Test_getS3Bucket_UploadChecksumHeaders` in `pkg/io/providers/aws_test.go` writes only five bytes, so it exercises the transfer manager's single-part `PutObject` path. The transfer manager's default multipart threshold is 16 MiB.

Write throwaway tests that exercise uploads larger than 16 MiB through `getS3Bucket` with `request_checksum_calculation` set to `when_required`, using a TLS `httptest` S3-compatible server and `insecure: true`, `s3_force_path_style: true`. Capture the `CreateMultipartUpload` and every `UploadPart` request. Assert that none has `Content-Encoding: aws-chunked` or a non-empty `X-Amz-Trailer`. Inspect request checksum and signature headers as needed to distinguish a real regression from an invalid test expectation.

Run the probe tests. For any failure, classify it as a real implementation bug or a test mistake, with evidence. If a real bug is found, apply the minimal implementation fix in `pkg/io/providers/aws.go` and related code only if strictly necessary, then rerun. Do not submit the throwaway tests. Delete them before finishing; leave only the minimal production fix, if any.

Acceptance criteria: the multipart path is exercised for `when_required`; `CreateMultipartUpload` and every `UploadPart` request are checked for chunked encoding and trailers; each failure is classified with evidence; if no real bug is found, no files are changed; if a real bug is found, its failing input, expected result, and actual result are reported and the minimal fix is applied.

Scope guard: do not refactor checksum configuration, change default behavior, alter the existing single-part regression test, add permanent tests, or change unrelated S3 provider behavior.
```
