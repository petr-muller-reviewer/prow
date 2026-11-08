---
pr: kubernetes-sigs/prow#929
title: "Added a case for tarball files on gcs"
head_sha: be2354fe7245b1629b9ab84aff381342ae798784
base: main
reviewed_at: 2026-11-07T23:59:02Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-09-10T12:10:50Z
  gated_head_sha: be2354fe7245b1629b9ab84aff381342ae798784
  reviewed_head_sha: be2354fe7245b1629b9ab84aff381342ae798784
refresh_log:
  - at: 2026-11-07T23:59:02Z
    old_sha: be2354fe7245b1629b9ab84aff381342ae798784
    new_sha: be2354fe7245b1629b9ab84aff381342ae798784
    summary: "No code changes. PR merged at 2026-09-10T12:20:34Z after petr-muller approved head be2354f; approval-notifier bot comment recorded. Findings unchanged."
---

## Gate

**Decision: merge.** This PR was merged on 2026-09-10T12:20:34Z at head `be2354fe7245b1629b9ab84aff381342ae798784`; no code changed after the review. There are no unresolved blocking or should-fix findings. The object-key and HTTP-representation change is backward-incompatible for consumers that depend on the old transformed `.tar` path or metadata, but it is the intentional substance of the fix, is explicitly disclosed in the PR description, and restores the filename and bytes produced by the job. `Prucek` approved the current head; `dtantsur` independently confirmed the current behavior causes real KDE-user friction.

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

Since previous review:

- No new commits: head is unchanged at `be2354fe7245b1629b9ab84aff381342ae798784`.
- `petr-muller` submitted an `APPROVED` review on 2026-09-10T12:18:29Z, followed by the `[APPROVALNOTIFIER]` bot comment at 12:18:36Z.
- The PR was merged on 2026-09-10T12:20:34Z.

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

## Followups

Recorded 2026-11-08 against merged PR #929 (merge commit `e30e351eb6310dda4deffe17edac616debcc13a9`). 1 accepted as probe, 4 skipped.

### probe: probe the artifact upload path for `.tar.gz` keys and options (`pkg/gcsupload/run.go`, `pkg/pod-utils/gcs`)
- mode: probe
- necessity: should
- why: `pkg/pod-utils/gcs/metadata_test.go:109-135` pins the pure transformation, but nothing exercises the uploaded object key, so a regression in `gatherArtifacts`/`escapeFileName` or in the compression writer would go unnoticed. The probe exercises the four paths where a `.tar.gz` artifact can be routed (artifact-dir walk, direct `Items`, sibling `.tar`, `compress_file_types`), where the pre-merge code collapsed keys and a mis-set `Content-Encoding` would silently corrupt the download.
- skipped: `compress_file_types` gzip-encoding plain `.tar` artifacts (considered by design, opt-in); stale `WriterOptionsFromFileName` doc comment; case-insensitive `.tar` suffix matching; test-only `.tar` MIME registration.

```text
In kubernetes-sigs/prow, probe the artifact upload path for compressed tarballs, following merged
PR #929 "Added a case for tarball files on gcs" (merge commit
e30e351eb6310dda4deffe17edac616debcc13a9). This is a probe: write throwaway tests, hunt for real
bugs, and keep only the minimal fix if one is found.

Start from the merged default branch (`main`). Relevant code: `pkg/pod-utils/gcs/metadata.go`
(`WriterOptionsFromFileName`), `pkg/pod-utils/gcs/upload.go` (`Upload`, `shouldCompressFileType`,
`openerObjectWriter.Write`), and `pkg/gcsupload/run.go` (`AssembleTargets`, `gatherArtifacts`,
`escapeFileName`). Model the tests on `pkg/pod-utils/gcs/metadata_test.go:109-135` and
`pkg/gcsupload/run_test.go:34` (`TestOptions_AssembleTargets`), and use `pkg/pod-utils/gcs/upload_test.go`
(`TestShouldCompressFileType`, `Test_openerObjectWriter_Write`, fakestorage) for the writer side.

Exercise at least these behaviours:
1. An artifact directory containing `must-gather.tar.gz` and `must-gather.tar.gzip` produces upload
   keys that keep the full suffix, with `Content-Type: application/gzip` and no `Content-Encoding`
   (before the PR both would have landed at `must-gather.tar` with `Content-Encoding: gzip`).
2. The same directory also containing a plain `must-gather.tar` sibling uploads all three without one
   shadowing another (the pre-merge code sent the compressed tarball to the plain `.tar` key, so the
   duplicate-upload warning fired and a file was silently dropped).
3. `compress_file_types: ["*"]` in `GCSConfiguration` does not double-compress a `.tar.gz` artifact
   (see the `ext == "gz" || ext == "gzip"` early return in `shouldCompressFileType`) and does not add
   `Content-Encoding: gzip` to the writer options.
4. A file named `foo.tar.gz` passed directly through `options.Items` (the non-directory branch of
   `AssembleTargets`, `pkg/gcsupload/run.go:139`) keeps its key just like the artifact-dir walk case.

Triage every failure: real implementation bug, or wrong expectation / test mistake, with evidence
(the observed key, `Content-Type`, or writer options). A real bug means the object key,
`Content-Type`, or `Content-Encoding` differs from what the PR promises consumers, or an upload is
silently dropped.

Acceptance criteria: all four behaviours are exercised and every failure is classified with evidence.
If a real bug is found, apply the minimal fix plus the smallest regression test that pins it. If none
is found, leave the tree unmodified.

Scope guard: do not redesign `WriterOptionsFromFileName`. Case-insensitive suffix matching, explicit
`.tar` MIME registration, and whether `compress_file_types` should skip plain `.tar` inputs were
reviewed and deliberately deferred or declined — out of scope. Delete the probe tests before
finishing; the deliverable is a bug fix, never new test files. Do not touch unrelated packages or docs.
```
