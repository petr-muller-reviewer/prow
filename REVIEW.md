---
pr: kubernetes-sigs/prow#949
title: "chore(deps): bump cloud.google.com/go/storage from 1.66.0 to 1.67.1"
head_sha: 8332d2a2b2af19cb14eed2f20e67dd4dbe67e255
base: main
reviewed_at: 2026-11-13T17:26:00Z
verdict: approve
refresh_log:
  - at: 2026-11-13T17:26:00Z
    old_sha: 8332d2a2b2af19cb14eed2f20e67dd4dbe67e255
    new_sha: 8332d2a2b2af19cb14eed2f20e67dd4dbe67e255
    summary: "No code changes; incorporated the 2026-09-22 approval, APPROVALNOTIFIER comment, and merge."
---

## Verdict

approve — dependency-only update with no findings. The tagged release is 13 days old; its two storage changes are unused by Prow. `go test ./pkg/io` passes. The PR was merged as `8332d2a` on 2026-09-22 with no changes after this review.

## What this PR does

- Updates the direct Go module `cloud.google.com/go/storage` from `v1.66.0` to `v1.67.1`.
- Updates the corresponding two checksums in `go.sum`.
- Includes storage v1.67.0, which promotes gRPC bidi-read and appendable-upload options from experimental.
- Includes storage v1.67.1, which prevents a nil-callback panic in `MultiRangeDownloader`.

Since previous review (2026-09-21):
- No code changes: the PR head is still `8332d2a`; nothing was pushed after the review.
- `petr-muller` approved on 2026-09-22T22:02:13Z and APPROVALNOTIFIER recorded the approval (`kubernetes-prow[bot]`, 2026-09-22T22:02:21Z).
- The PR was merged into `main` on 2026-09-22T22:22:20Z.

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

## Dependency followups

No opportunities identified — reviewed 2026-11-13 (PR #949 merged as `bbd8955`).

- `cloud.google.com/go/storage` v1.66.0 → v1.67.1 (direct): the only storage changes in range are the v1.67.0 promotion of `WithGRPCBidiReads` and appendable uploads out of the `storage/experimental` package (a relocation, plus `experimental.WithGRPCAppendableUploads` becoming `storage.WithAppendableUploads()`) and the v1.67.1 nil-callback panic fix in `MultiRangeDownloader`. Prow imports none of `storage/experimental`, `WithGRPCBidiReads`, `WithZonalBucketAPIs`, `WithGRPCAppendableUploads`/`WithAppendableUploads`, `MultiRangeDownloader`, or `NewGRPCClient`; its GCS surface in `pkg/io` (`NewClient`, `BucketHandle`, `ObjectHandle`, `Conditions`, `ObjectAttrsToUpdate`, `SignedURL`, `Query`, `Writer`, `ErrObjectNotExist`) is untouched by the release and carries no new deprecation.
- No other direct or transitive module changed in this bump.
