---
pr: kubernetes-sigs/prow#976
title: "fix(hack): make verify-gofmt read-only and fail on unformatted code"
head_sha: b48dea0bea696cf2acf7bfbf867ddc18d48741ff
base: main
reviewed_at: 2026-09-27T12:01:00Z
verdict: approve
---

# Review

## Verdict

Approve.

The change implements the behavior requested in issue #953. The verification target now reports unformatted Go files without rewriting them, and the three pre-existing formatting violations are fixed. I found no actionable standards or spec issues.

## What this PR does

- Replaces the verifier's write-and-diff sequence with `gofmt -s -l`.
- Returns failure and points contributors to `make update-gofmt` when files need formatting.
- Applies formatting-only changes to the three files named in issue #953.

## Findings

None.

## Resolved

None.

## Checked

- Compared the patch with [issue #953](https://github.com/kubernetes-sigs/prow/issues/953) and the PR description; all requested changes are present.
- `git diff --check upstream/main...HEAD` passed.
- `make verify-gofmt` passed on the clean tree.
- With a temporary unformatted Go file, `make verify-gofmt` failed, printed the file path and `make update-gofmt`, and left the file byte-for-byte unchanged. The temporary file was removed.
- The `make update-gofmt` target and its script remain unchanged.

## Open questions

None.

## Followups

### Make `verify-codegen` read-only

- Category: cleanup
- Where: `hack/make-rules/verify/codegen.sh:29-52` and `hack/make-rules/update/codegen.sh`
- Necessity: should
- Why followup: PR #976 makes the gofmt verifier read-only, but `verify-codegen` still runs the update script in the checkout and restores only some generated paths afterward. This is a separate verifier change and does not block PR #976.

```text
In kubernetes-sigs/prow, after PR #976 ("fix(hack): make verify-gofmt read-only and fail on unformatted code") merges, make hack/make-rules/verify/codegen.sh read-only with respect to tracked source files. It currently runs hack/make-rules/update/codegen.sh in the checkout, then attempts to restore selected paths; the restore omits pkg/gangway and does not run if generation fails. Run generation in an isolated copy instead, compare every output produced by update/codegen.sh against the checkout, and report stale generated files with the existing make update-codegen remedy.

Acceptance criteria: a clean checkout passes; a deliberately stale generated output makes verification fail with a useful diagnostic; tracked files in the original checkout are byte-for-byte unchanged after successful verification, stale-output failure, and a generator error. Cover all generated output locations used by update/codegen.sh, and clean up temporary files on every exit path. Keep the change scoped to codegen verification and any focused tests or helpers it needs; do not alter gofmt verification or unrelated generator behavior.
```
