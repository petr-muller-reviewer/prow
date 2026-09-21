---
pr: kubernetes-sigs/prow#949
title: "chore(deps): bump cloud.google.com/go/storage from 1.66.0 to 1.67.1"
head_sha: 8332d2a2b2af19cb14eed2f20e67dd4dbe67e255
base: main
reviewed_at: 2026-09-21T22:55:57Z
verdict: approve
---

## Verdict

approve — dependency-only update with no findings. The tagged release is 13 days old; its two storage changes are unused by Prow. `go test ./pkg/io` passes.

## What this PR does

- Updates the direct Go module `cloud.google.com/go/storage` from `v1.66.0` to `v1.67.1`.
- Updates the corresponding two checksums in `go.sum`.
- Includes storage v1.67.0, which promotes gRPC bidi-read and appendable-upload options from experimental.
- Includes storage v1.67.1, which prevents a nil-callback panic in `MultiRangeDownloader`.

## Findings

No findings.

## Checked

- Classification: only `go.mod` and `go.sum` change; no project source, tests, or generated code change.
- Direct usage: seven importing Go files; production use is confined to `pkg/io/option.go`, `pkg/io/opener.go`, and `pkg/io/iterator.go`.
- Exposure: Prow's use covers GCS clients, credentials, object I/O, listing, and signed URLs, but does not use `WithGRPCBidiReads`, `WithGRPCAppendableUploads`, or `MultiRangeDownloader`.
- Release provenance: `v1.67.1` is a normal tag from 2026-09-08, 13 days before review; no published repository security advisory or listed CVE applies to this range.
- `git diff --check` passes.
- `go test ./pkg/io` passes.

## Open questions

None.
