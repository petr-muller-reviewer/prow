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
