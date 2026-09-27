---
issue: kubernetes-sigs/prow#953
title: "verify-gofmt cannot fail: it runs gofmt -w before the check"
state: closed
labels: kind/bug
main_sha: cd1c1dbd246180e7183af5eae6cc68c2053ffcdc
triaged_at: 2026-09-27T13:09:11Z
verdict: resolved
refresh_log:
  - since: 2026-09-22T00:25:16Z
    summary: "PR #976 merged and closed the issue; the read-only verifier addresses the triaged cause."
---

## Verdict

Resolved by PR #976. The merged verifier lists unformatted files without writing them and fails when the list is nonempty.

## Resolution

PR #976 merged at `2026-09-27T12:22:37Z`; GitHub closed the issue as `COMPLETED` at `2026-09-27T12:22:38Z`. Classification: **sufficient**.

- `hack/make-rules/verify/gofmt.sh` replaces `gofmt -s -w` followed by `gofmt -s -d` with `gofmt -s -l`, then exits 1 when files are listed. This addresses the identified cause and leaves `update-gofmt` as the write path.
- The PR formats the same three baseline files found during triage, so the new check can pass on the merged tree.
- The PR reports a failing malformed-file check and a passing clean-tree check with no writes. No comments after the merge report a remaining failure. This matches the previously recommended fix and command-level verification.

## What the issue reports

- `make verify-gofmt` passes malformed Go code after rewriting it.
- `make verify` includes the same mutating check.
- `update-gofmt` should retain write behavior; verification should only report failure.
- Three tracked files were listed by a non-mutating formatter scan at the recorded main SHA.

Since previous triage:

- PR #976 was cross-referenced on `2026-09-25T09:22:40Z`, merged on `2026-09-27`, and closed the issue.
- The merged change makes verification read-only and formats the three previously identified files.

## Findings

### [reproducibility] Baseline formatting drift is observable without writes
- detail: `gofmt -s -l` lists `cmd/checkconfig/main.go`, `pkg/pjutil/filter_test.go`, and `pkg/plugins/rifle/rifle_test.go` in the checked-out main SHA.
- evidence: non-mutating formatter listing at `cd1c1dbd246180e7183af5eae6cc68c2053ffcdc`.

### [cause] Verifier writes before it checks
- detail: The first command formats all Go files in place. The second command therefore observes no formatting diff and the following empty-diff branch exits zero unless gofmt itself errors.
- evidence: `hack/make-rules/verify/gofmt.sh:26-30`.

### [related-code] Updater owns the mutation
- where: `hack/make-rules/update/gofmt.sh:26-27`
- excerpt: |
    echo "Go version: $(go version)"
    find . -name '*.go' -type f -print0 | xargs -0 gofmt -s -w
- relevance: This is the correct location for formatting writes.

### [related-code] Make targets distinguish update from verify
- where: `Makefile:91-96`
- excerpt: |
    update-gofmt:
        hack/make-rules/update/gofmt.sh
    verify-gofmt:
        hack/make-rules/verify/gofmt.sh
- relevance: The target structure supports a non-mutating verifier.

### [related-code] Aggregate verification invokes the faulty script
- where: `hack/make-rules/verify/all.sh:35-39`
- excerpt: |
    if [[ "${VERIFY_GOFMT:-true}" == "true" ]]; then
      name="go fmt"
      hack/make-rules/verify/gofmt.sh || { FAILED+=($name); echo "ERROR: $name failed"; }
    fi
- relevance: The defect affects `make verify`, not only the standalone target.

### [related-pr] No existing fix at initial triage
- relevance: At the initial triage, repository PR search found no active or historical PR specifically addressing this verifier defect.

### [related-pr] Merged verifier fix
- ref: kubernetes-sigs/prow#976
- relevance: Replaces the mutating check with a read-only file listing and formats the three baseline files.

## Checked

- Fetched the complete open issue and its only bot comment.
- Inspected the verifier, updater, Makefile targets, and aggregate verification path at `cd1c1dbd246180e7183af5eae6cc68c2053ffcdc`.
- Confirmed the control flow and formatter listing without mutating tracked files.
- Confirmed `kind/bug`, `sig/testing`, and `good first issue` exist; `area/prow` does not.
- Reviewed PR #976's description, changed files, and diff against the recorded cause and reproducibility finding; checked current issue state, comments, and timeline through `2026-09-27T13:09:11Z`.

## Next steps

- No further issue triage action. Optional post-merge verification: run `make verify-gofmt` on the merged main branch and confirm the tree stays clean.

## Open questions

- None for this resolved issue.
