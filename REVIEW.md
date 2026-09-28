---
pr: kubernetes-sigs/prow#734
title: "chore: upgrade golangci-lint to v2.12.2 and fix new lint issues"
head_sha: d50fe2b97211aeb543d61aa9eb18549ac658edb1
base: main
reviewed_at: 2026-05-28T23:35:55Z
verdict: request-changes
---

## Findings

### [should-fix] slices.Backward with in-place slice mutation in DeleteComment
- where: `pkg/tide/tide_test.go:923-927`
- concern: The original manual backward loop was safe for in-place deletion (removing element `j` while decrementing doesn't shift unvisited lower indices). `slices.Backward()` constructs an iterator from the original slice; mutating the backing array via `append(ics[:j], ics[j+1:]...)` during iteration can cause skipped or duplicated elements. The missing `break` after mutation (unlike the `lgtm.go` version) makes this worse. Either revert to the manual loop, use `slices.DeleteFunc`, or add a `break`.
- excerpt: |
    for j, v := range slices.Backward(ics) {
        if v.ID == id {
            f.issueComments[issue] = append(ics[:j], ics[j+1:]...)
        }
    }

### [nit] Redundant intermediate variable in lgtm.go
- where: `pkg/plugins/lgtm/lgtm.go:458-459`
- concern: `comment := v` is unnecessary; the range variable can be named `comment` directly: `for _, comment := range slices.Backward(comments)`.
- excerpt: |
    for _, v := range slices.Backward(comments) {
        comment := v

## Checked
- All six `reflect.Ptr` to `reflect.Pointer` replacements in `pkg/genyaml/genyaml.go` and `pkg/genyaml/populate_struct.go` — correct alias swaps, same constant value since Go 1.18
- `slices.Backward` in `pkg/plugins/lgtm/lgtm.go:458` — correct, loop breaks on first match, no mutation during iteration
- `hack/tools/go.mod` version bump v2.11.3 to v2.12.2 and indirect dependency churn — mechanical, expected, isolated to `hack/tools/` module
- `hack/tools/go.sum` — consistent with go.mod changes
- No configuration, API, CLI flag, or behavioral changes — zero deployment risk
- Go version requirement: `slices.Backward` needs Go 1.23+, module already declares `go 1.25.5`

## Open questions
- In `tide_test.go:DeleteComment`, was there a reason not to add a `break` after the slice mutation? The `lgtm.go` version breaks on first match.
- In `lgtm.go:458-459`, was there a reason to keep the intermediate `comment := v` assignment rather than naming the range variable `comment` directly?
