---
issue: kubernetes-sigs/prow#953
title: "verify-gofmt cannot fail: it runs gofmt -w before the check"
state: open
labels: kind/bug
main_sha: cd1c1dbd246180e7183af5eae6cc68c2053ffcdc
triaged_at: 2026-09-22T00:25:16Z
verdict: accepted
---

## Verdict

Accepted. The checked-in verifier formats every Go file before computing its diff, so normal formatter execution always reaches the empty-diff success branch and mutates the working tree.

## What the issue reports

- `make verify-gofmt` passes malformed Go code after rewriting it.
- `make verify` includes the same mutating check.
- `update-gofmt` should retain write behavior; verification should only report failure.
- Three tracked files are presently listed by a non-mutating formatter scan.

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

### [related-pr] No existing fix found
- relevance: Repository PR search found no active or historical PR specifically addressing this verifier defect.

## Checked

- Fetched the complete open issue and its only bot comment.
- Inspected the verifier, updater, Makefile targets, and aggregate verification path at `cd1c1dbd246180e7183af5eae6cc68c2053ffcdc`.
- Confirmed the control flow and formatter listing without mutating tracked files.
- Confirmed `kind/bug`, `sig/testing`, and `good first issue` exist; `area/prow` does not.

## Next steps

- Retain `kind/bug`; optionally add `sig/testing` and `good first issue`.
- Remove the write command from `hack/make-rules/verify/gofmt.sh`, preserving its diff-based failure diagnostic.
- Format the three baseline files in the same PR with `make update-gofmt` or equivalent.
- Demonstrate malformed temporary input fails without mutation; then run `make verify-gofmt` and `git diff --exit-code` on the clean tree.

## Open questions

- Should a focused regression test be added for this make-rule, or is command-level verification sufficient?
- Do maintainers prefer the existing diff diagnostic or a file-list-only `gofmt -l` diagnostic?
