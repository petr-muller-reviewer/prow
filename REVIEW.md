---
pr: kubernetes-sigs/prow#971
title: "Avoid aws-chunked 501 error on S3-compatible stores"
head_sha: 33bee48b0d9c19b96170dbb63c9a7b9af7d1d70e
base: main
reviewed_at: 2026-10-19T12:57:18Z
verdict: request-changes
refresh_log:
  - old_sha: 4dc604d56097e0566f9138475e12abd463284586
    new_sha: 33bee48b0d9c19b96170dbb63c9a7b9af7d1d70e
    summary: "The PR now makes request checksum mode configurable, but the Go CDK upload path is unchanged and the blocking finding remains."
gate:
  decision: do-not-merge
  gated_at: 2026-10-19T13:08:59Z
  gated_head_sha: 33bee48b0d9c19b96170dbb63c9a7b9af7d1d70e
  reviewed_head_sha: 33bee48b0d9c19b96170dbb63c9a7b9af7d1d70e
---

# Review

## Gate

**Decision: do-not-merge.** PR #971 is open at `33bee48b`, the same head recorded in this review. Its blocking finding remains: the configured client checksum mode does not prevent the Go CDK transfer manager from sending `aws-chunked` on the actual upload path.

### Gating items

- **Not addressed — blocking (`REVIEW.md`, `pkg/io/providers/aws.go:48-49,79-89`):** `newS3Client` applies `request_checksum_calculation`, but `getS3Bucket` still calls `s3blob.OpenBucketV2(ctx, client, bucketName, nil)`. With the pinned Go CDK transfer manager, the writer explicitly selects CRC32 for `PutObject`, overriding this client setting; the request-level reproduction recorded above still applies. The added tests inspect client options and do not send an upload request. **Disposition:** blocks merge until an upload through `providers.GetBucket` omits the optional checksum trailer and a request-level test verifies it.

### Independent merge risk

- **Additive exported Go API:** `pkg/io/providers.S3Credentials` gains the exported `RequestChecksumCalculation` field. JSON users and keyed struct literals remain compatible, but external Go callers using unkeyed composite literals of this type will need to add a value. The field is documented in code and is opt-in; an empty value preserves the SDK and environment policy, so existing deployments do not get a new checksum default. This source-compatibility edge is not independently merge-blocking.
- No other default or configuration behavior changes were found in the PR diff. No substantive PR reviews or inline comments were submitted; the issue comments are bot notifications.

## Verdict

Request changes: the upload still sends `aws-chunked` to custom S3 endpoints.

The new setting is present on the S3 client, but the Go CDK upload path uses a transfer manager that explicitly selects CRC32. A request-level reproduction through `providers.GetBucket` still produced the checksum trailer and the reported `501 NotImplemented`. The new unit tests inspect client options and therefore cannot catch this failure.

## What this PR does

- Adds the optional `request_checksum_calculation` S3 credential field, accepting `when_supported` or `when_required` and rejecting other values.
- Applies an explicit field value to the AWS SDK client; when unset, preserves the SDK default and `AWS_REQUEST_CHECKSUM_CALCULATION` behavior.
- Documents `when_required` for S3-compatible stores and adds client-option tests for field and environment precedence.

Since previous review:

- Replaced the automatic custom-endpoint default with an opt-in JSON setting and added validation and an OCI/Swift example (`pkg/io/providers/aws.go`, `pkg/io/providers/providers.go`).
- Expanded the client configuration tests (`pkg/io/providers/aws_test.go`). The Go CDK bucket-opening and transfer-manager upload path did not change, so the prior blocking finding remains unresolved.

## Findings

### [blocking] The actual upload still uses aws-chunked encoding

- where: `pkg/io/providers/aws.go:48-49,79-89`
- concern: The new `request_checksum_calculation` field sets the SDK client mode, but `getS3Bucket` still opens Go CDK's S3 bucket with nil options. With the pinned `gocloud.dev v0.46.0` and `feature/s3/transfermanager v0.2.3`, its writer explicitly selects CRC32 for `PutObject`, overriding this client setting. The prior local TLS reproduction observed `Content-Encoding: aws-chunked`, `X-Amz-Trailer: x-amz-checksum-crc32`, and `X-Amz-Sdk-Checksum-Algorithm: CRC32`, then returned the same `501 NotImplemented` described in [issue #970](https://github.com/kubernetes-sigs/prow/issues/970). This commit changes only client configuration, tests, and documentation; it does not address the upload path. The [AWS transfer-manager fix](https://github.com/aws/aws-sdk-go-v2/pull/3470) and [Go CDK propagation fix](https://github.com/google/go-cloud/pull/3766) remain absent from the pinned versions.
- excerpt: |
    bkt, err := s3blob.OpenBucketV2(ctx, client, bucketName, nil)
    }

## Checked

- `go test ./pkg/io/providers -run 'Test_newS3Client' -count=1` passed.
- The new unit cases assert S3 client option values, field validation, and environment precedence; they do not send an upload request.
- A streamed upload through `providers.GetBucket` to a local TLS server reproduced the `aws-chunked` headers and simulated `501` failure.
- The new field is opt-in and unset preserves the SDK/environment policy; setting it does not alter the Go CDK bucket writer's checksum selection.

## Open questions

- Could you update the upload path so that the transfer manager receives `when_required` for custom endpoints, then add a request-level test asserting that the resulting `PutObject` omits `aws-chunked`?
