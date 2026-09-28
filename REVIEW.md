---
pr: kubernetes-sigs/prow#734
title: "chore: upgrade golangci-lint to v2.12.2 and fix new lint issues"
head_sha: d50fe2b97211aeb543d61aa9eb18549ac658edb1
base: main
reviewed_at: 2026-09-28T20:12:00Z
verdict: request-changes
refresh_log:
  - old_sha: d50fe2b97211aeb543d61aa9eb18549ac658edb1
    new_sha: d50fe2b97211aeb543d61aa9eb18549ac658edb1
    summary: "No new commits; recorded approval, approved label, and merge on 2026-06-02."
---

## What this PR does

- Upgrades `golangci-lint` from v2.11.3 to v2.12.2 in `hack/tools/`.
- Updates Go code in `pkg/genyaml/`, `pkg/plugins/lgtm/`, and `pkg/tide/` to satisfy the new lint checks.

Since previous review:

- No code changes; the head remains `d50fe2b97211aeb543d61aa9eb18549ac658edb1`.
- petr-muller approved the PR on 2026-06-02 at 16:46:06 UTC; k8s-ci-robot posted the approval notice and added the `approved` label shortly afterward.
- The PR was merged on 2026-06-02 at 17:15:49 UTC. No new inline review comments were posted. The findings below remain as recorded at the original review.

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

## Followups

### Handle GitHub read failures in LGTM tree-hash checks
- category: reliability
- where: `pkg/plugins/lgtm/lgtm.go:350-376,452-475`; `pkg/plugins/lgtm/lgtm_test.go`
- necessity: should — a failed comment lookup can remove LGTM, while failed pull-request or commit lookups can leave partial label changes or post an empty tree hash.
- why followup: PR #734 touched the tree-hash scan while upgrading the linter; the error handling predates the PR and did not block its merge.
- handoff prompt:

```text
In kubernetes-sigs/prow, following merged PR #734 ("chore: upgrade golangci-lint to v2.12.2 and fix new lint issues", merge commit 52c1eeb13bd2f1a241b2314b5ca06ed55ab17b2e), make LGTM tree-hash handling fail safely when GitHub reads fail. Work on the merged default branch. Inspect pkg/plugins/lgtm/lgtm.go, especially handlePullRequest's ListIssueComments/GetSingleCommit path and handle's GetPullRequest/GetSingleCommit path, and add focused cases in pkg/plugins/lgtm/lgtm_test.go.

Return contextual errors when these reads fail. Arrange the reads so those failures do not remove or add an LGTM label, request review, or post a tree-hash comment; never post a tree-hash comment with an empty hash after a failed read. Cover each failure path with tests that assert the error and absence of those side effects, while preserving the successful tree-hash behavior.

Keep the change limited to tree-hash read/error handling and its tests. Do not update golangci-lint or redesign unrelated label handling.
```
