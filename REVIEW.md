---
pr: kubernetes-sigs/prow#971
title: "Avoid aws-chunked 501 error on S3-compatible stores"
head_sha: 4dc604d56097e0566f9138475e12abd463284586
base: main
reviewed_at: 2026-09-27T14:19:33Z
verdict: request-changes
---

# Review

## Verdict

Request changes: the upload still sends `aws-chunked` to custom S3 endpoints.

The new setting is present on the S3 client, but the Go CDK upload path uses a transfer manager that explicitly selects CRC32. A request-level reproduction through `providers.GetBucket` still produced the checksum trailer and the reported `501 NotImplemented`. The new unit tests inspect client options and therefore cannot catch this failure.

## What this PR does

- Sets the S3 client's request checksum policy to `when_required` when credentials specify a custom endpoint.
- Leaves the SDK-loaded policy in place when `AWS_REQUEST_CHECKSUM_CALCULATION` is set or no custom endpoint is specified.
- Adds three unit cases covering those client-option combinations.

## Findings

### [blocking] The actual upload still uses aws-chunked encoding

- where: `pkg/io/providers/aws.go:79-80`
- concern: `getS3Bucket` opens Go CDK's S3 bucket with nil options at `pkg/io/providers/aws.go:49`. With the pinned `gocloud.dev v0.46.0` and `feature/s3/transfermanager v0.2.3`, its writer makes the transfer manager explicitly select CRC32 for `PutObject`; that overrides the effect of this client setting. A local TLS server receiving an upload through `providers.GetBucket` observed `Content-Encoding: aws-chunked`, `X-Amz-Trailer: x-amz-checksum-crc32`, and `X-Amz-Sdk-Checksum-Algorithm: CRC32`, then returned the same `501 NotImplemented` described in [issue #970](https://github.com/kubernetes-sigs/prow/issues/970). The [AWS transfer-manager fix](https://github.com/aws/aws-sdk-go-v2/pull/3470) and [Go CDK propagation fix](https://github.com/google/go-cloud/pull/3766) are upstream, but neither is in the versions pinned here.
- excerpt: |
    if creds.Endpoint != "" && os.Getenv("AWS_REQUEST_CHECKSUM_CALCULATION") == "" {
        cfg.RequestChecksumCalculation = aws.RequestChecksumCalculationWhenRequired
    }

## Checked

- `go test ./pkg/io/providers -run 'Test_newS3Client' -count=1` passed.
- The three new cases assert the intended S3 client option values; they do not send an upload request.
- A streamed upload through `providers.GetBucket` to a local TLS server reproduced the `aws-chunked` headers and simulated `501` failure.
- The change leaves the client checksum setting unchanged for ordinary AWS S3 endpoints and preserves an explicit environment setting at config load time.

## Open questions

- Could you update the upload path so that the transfer manager receives `when_required` for custom endpoints, then add a request-level test asserting that the resulting `PutObject` omits `aws-chunked`?
