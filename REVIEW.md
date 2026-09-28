---
pr: kubernetes-sigs/prow#683
title: "Fix expanding skipped lines when using S3"
head_sha: 61a57d1fb4cd181461577c9ef1144239356b23ea
base: main
reviewed_at: 2026-09-28T20:45:18Z
verdict: approve
---

# Review

## Verdict

Approve with suggestions. The change fixes S3 range reads that return bytes and `io.EOF` together, and the focused test passes. Code quality and maintainability reviewers recommend approval; deployment risk is low. The suggestions below are non-blocking.

## What this PR does

- Accepts `io.EOF` from a range reader once `ReadAt` has filled the requested buffer, including ranges inside an artifact.
- Reports `io.ErrUnexpectedEOF` when the range reader stops before filling the buffer.
- Simulates S3's final-read behavior in the test helper and adds a test for an interior range.

## Findings

### [nit] Keep the test helper's reader type explicit

- where: `pkg/spyglass/storageartifact_test.go:55-56`
- concern: The new type assertion assumes the embedded `io.Reader` is a `*bytes.Reader`. Every current construction satisfies that assumption, but extending the helper with another reader would panic when `returnEOF` is set. A concrete reader field or explicit remaining-byte tracking would make the test helper's requirement clear.
- excerpt: |
    if rc.returnEOF && read > 0 && rc.Reader.(*bytes.Reader).Len() == 0 {
        return read, io.EOF
    }

### [nit] Cover EOF before the requested range is full

- where: `pkg/spyglass/storageartifact_test.go:321-328`
- concern: The added case covers EOF after a complete interior range. A case where the range reader returns EOF after fewer bytes would preserve the new `io.ErrUnexpectedEOF` behavior against future changes.
- excerpt: |
    name:      "ReadAt S3-style EOF (EOF when range finished but not at end of file)",
    n:         4,
    offset:    6,
    returnEOF: true,

### [nit] Explain the range reader EOF condition

- where: `pkg/spyglass/storageartifact.go:169-173`
- concern: A short comment would make it clear why EOF can be accepted when the requested range ends before the artifact does. This behavior is surprising without the PR description.
- excerpt: |
    if err == io.EOF {
        if offset == len(p) {
            break
        }
        err = io.ErrUnexpectedEOF
    }

## Checked

- Spec: The PR description's start and middle skipped-line failure is addressed by accepting EOF after the requested range is full, independent of the range's position. The test covers a middle range; the same branch handles a range starting at offset zero.
- Spec: `ReadAt` still returns `io.EOF` for a range ending at the overall artifact end, and an early EOF now signals an incomplete read.
- Standards: `CONTRIBUTING.md` has no coding rule specific to this change; no documented-standard violation was found.
- Code quality: No correctness, safety, or performance issue was found in the changed code.
- Deployment risk: No configuration, API, permission, dependency, rollout, or migration change is required; the behavioral change is limited to Spyglass range reads.
- Maintainer review: No concern was raised by multiple reviewers, and none of the suggestions is required before merge.
- `go test ./pkg/spyglass -run '^TestReadAt$' -count=1` passed.

## Open questions

None.

## Followups

No followups accepted. Two candidates were skipped: a premature-EOF regression test and returning the byte count on a partial read error.
