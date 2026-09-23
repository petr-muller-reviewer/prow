---
pr: kubernetes-sigs/prow#783
title: "repoowners: add advisory_approvers OWNERS field"
head_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
base: main
reviewed_at: 2026-09-21T22:37:17Z
verdict: request-changes
---

## What this PR does

- Adds `advisory_approvers` to OWNERS configuration, preserving /approve authority while excluding advisory users from auto-assignment.

## Findings

### [blocking] LeafApprovers returns empty set for advisory-only directories
- where: `pkg/repoowners/repoowners.go:903-908`
- concern: If a directory's OWNERS lists only `advisory_approvers` with no regular `approvers`, `applyConfigToPath` merges advisory users into `o.approvers` at that path. `entriesForFile` with `leafOnly: true` finds them and stops, never reaching parent. The subtraction yields empty. Auto-assignment finds zero candidates.
- excerpt: |
    all := o.entriesForFile(path, o.approvers, true).Set()
    advisory := o.entriesForFile(path, o.advisoryApprovers, true).Set()
    return all.Difference(advisory)

### [blocking] verify-owners rejects root OWNERS with only advisory_approvers
- where: `pkg/plugins/verify-owners/verify-owners.go:459-464`
- concern: `approvers` and `advisoryApprovers` are separate slices. The `len(approvers) == 0` root check fires before `advisoryApprovers` are appended to `owners` on line 466. A root OWNERS with only `advisory_approvers` is rejected as having no approvers.
- excerpt: |
    if filepath.Dir(c.Filename) == "." && len(approvers) == 0 {

### [should-fix] Dual-storage invariant is under-documented
- where: `pkg/repoowners/repoowners.go:270` and `:756-769`
- concern: Advisory approvers live in both `o.advisoryApprovers` and `o.approvers`. Only documented by comment in LeafApprovers. Future methods reading `o.approvers` must know advisory users are mixed in. Add doc comment on the `advisoryApprovers` struct field. Flagged by code quality and maintainability reviewers independently.

### [should-fix] TopLevelApprovers/LeafApprovers asymmetry undocumented
- where: `pkg/repoowners/repoowners.go:948-950`
- concern: `TopLevelApprovers` includes advisory approvers (reads `o.approvers` directly), `LeafApprovers` subtracts them. Asymmetry is correct but undocumented. A future contributor might "fix" it. Add one-line comment.

### [question] Is advisory-only OWNERS a valid use case?
- where: both blocking findings
- concern: Both blocking bugs only trigger when an OWNERS file has `advisory_approvers` but no regular `approvers`. If this is unsupported, add explicit validation. If supported, fix both bugs.

### [question] Should advisory approvers listed as reviewers appear in LeafReviewers?
- where: `pkg/repoowners/repoowners_test.go` TestAdvisoryApproverAlsoReviewer
- concern: Test asserts advisory approver who is also a reviewer still appears in LeafReviewers (can be auto-assigned as reviewer, not as approver). Confirm intentional.

## Checked
- RepoOwner no longer exposes AdvisoryApprovers; the six fake-only method implementations were removed
- LeafApprovers callers use it for assignment (excluding advisory is correct)
- Approvers callers use it for authorization (including advisory is correct)
- filterCollaborators preserves superset invariant (intersection preserves subsets)
- AllApprovers/AllOwners/TopLevelApprovers include advisory via merged o.approvers
- SimpleConfig.Empty() includes the new field
- Add-then-subtract design is safer default than keep-separate-union-in-Approvers
- Test coverage is thorough across repoowners and verify-owners
- Deployment risk is low: additive, omitempty, rolling upgrades/rollbacks safe

## Open questions
- Is advisory-only OWNERS (no regular approvers) a valid use case? Both blocking findings hinge on this.
- Should advisory approvers also listed as reviewers appear in LeafReviewers? TestAdvisoryApproverAlsoReviewer asserts yes.
