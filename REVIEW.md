---
pr: kubernetes-sigs/prow#929
title: "Added a case for tarball files on gcs"
head_sha: be2354fe7245b1629b9ab84aff381342ae798784
base: main
reviewed_at: 2026-09-09T11:01:57Z
verdict: approve
---

## Verdict

Approve with non-blocking follow-up suggestions. The tarball exception preserves the uploaded archive name and gzip bytes while retaining ordinary `.gz` content-encoding behavior. Unit coverage verifies the relevant object-name and metadata contracts.

## What this PR does

- Detects `.tar.gz` and `.tar.gzip` as archives rather than HTTP gzip-encoded files.
- Retains the full archive filename instead of stripping the compression suffix.
- Serves these archives as `application/gzip` with no `Content-Encoding`.
- Leaves existing gzip handling for logs and other files unchanged.

## Findings

No blocking, should-fix, nit, or question findings.

## Checked

- `pkg/pod-utils/gcs/metadata.go:42-69`: `.tar.gz` and `.tar.gzip` bypass `Content-Encoding: gzip`, preserve the filename, and receive `application/gzip`.
- `pkg/pod-utils/gcs/metadata.go:48-63`: non-tar gzip behavior still strips the encoding extension and derives the underlying MIME type.
- `pkg/pod-utils/gcs/metadata_test.go:113-135`: tests cover `.tar`, `.tar.gz`, and `.tar.gzip`, including their expected metadata.
- The externally visible object-key change is intentional and documented in the PR description.

## Open questions

- For rollout: do downstream artifact consumers, dashboards, and bucket policies that reference the historical `foo.tar` key tolerate the new `foo.tar.gz`/`foo.tar.gzip` names?

## Suggestions

- Consider extending the `WriterOptionsFromFileName` doc comment to state that tarball suffixes are intentionally preserved; this helps prevent a future simplification from removing the exception.
- Consider an end-to-end upload test that verifies the object name, unmodified gzip bytes, and absent `Content-Encoding`.
