---
pr: kubernetes-sigs/prow#929
title: "Added a case for tarball files on gcs"
head_sha: be2354fe7245b1629b9ab84aff381342ae798784
base: main
reviewed_at: 2026-09-10T12:10:32Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-09-10T12:10:50Z
  gated_head_sha: be2354fe7245b1629b9ab84aff381342ae798784
  reviewed_head_sha: be2354fe7245b1629b9ab84aff381342ae798784
---

## Gate

**Decision: merge.** There are no unresolved blocking or should-fix findings. The object-key and HTTP-representation change is backward-incompatible for consumers that depend on the old transformed `.tar` path or metadata, but it is the intentional substance of the fix, is explicitly disclosed in the PR description, and restores the filename and bytes produced by the job. `Prucek` approved the current head; `dtantsur` independently confirmed the current behavior causes real KDE-user friction.

Gating list:

- None.

Independent merge risks:

- `pkg/pod-utils/gcs/metadata.go:42-69`: future `.tar.gz`/`.tar.gzip` objects use a different key, MIME type, and content-encoding contract. Hard-coded object URLs, MIME-sensitive clients, and suffix-based bucket rules may need adjustment. This is acceptable because the compatibility break is documented and directly implements the requested behavior.
- The helper feeds the generic upload writer, so the behavior is not necessarily limited to GCS-backed deployments. The same filename-preservation contract is appropriate for other object stores, but this blast radius should remain visible in release communication.

## Verdict

Approve. The implementation correctly preserves compressed tarball names and stored gzip bytes while retaining the existing transparent gzip behavior for logs and other compressible files. Focused unit coverage exercises `.tar`, `.tar.gz`, and `.tar.gzip`.

## What this PR does

- Recognizes `.tar.gz` and `.tar.gzip` as downloadable compressed archives rather than HTTP-encoded tar streams.
- Retains the full archive filename instead of stripping the compression suffix.
- Stores these archives as `application/gzip` without `Content-Encoding`, preventing decompressive transcoding.
- Leaves existing `.gz` handling for logs and other files unchanged.

## Findings

### [question] Confirm downstream compatibility is acceptable
- where: `pkg/pod-utils/gcs/metadata.go:42-69`
- concern: Future objects move from `foo.tar` to `foo.tar.gz`/`foo.tar.gzip` and change from `application/x-tar` plus `Content-Encoding: gzip` to `application/gzip` without an encoding. Hard-coded URLs, clients that expect decompressed tar bytes, and suffix-based storage rules may need adjustment; this is non-blocking because the PR explicitly documents the key change and the changed representation is the intended fix.
- excerpt: |
    isTarball := isGzip && index > 0 && segments[index-1] == "tar"
    if !isTarball && segment != "" {
        if mediaType := mime.TypeByExtension("." + segment); mediaType != "" {
            attrs.ContentType = new(mediaType)
        }
    }
    if attrs.ContentType == nil && isGzip {
        attrs.ContentType = new("application/gzip")
        attrs.ContentEncoding = nil
    }

## Checked

- `pkg/pod-utils/gcs/metadata.go:42-69`: tarball suffixes preserve the filename and omit `Content-Encoding`; ordinary gzip inputs retain the previous suffix-stripping behavior.
- `pkg/pod-utils/gcs/metadata_test.go:109-135`: tests cover plain tar and both compressed-tar suffixes with their expected names and metadata.
- `go test ./pkg/pod-utils/gcs` passes.
- The PR description explicitly warns that future artifacts previously stored as `foo.tar` will be stored as `foo.tar.gz`.
- `dtantsur` reported on 2026-09-09 that KDE users currently need to rename every affected CI download manually.
- `Prucek` approved head `be2354fe7245b1629b9ab84aff381342ae798784` on 2026-09-09.

## Open questions

- Have known consumers that construct archive paths directly, rather than listing artifacts, been considered for the `foo.tar` to `foo.tar.gz` transition? This does not block merge, but the compatibility note should be retained in release communication.
