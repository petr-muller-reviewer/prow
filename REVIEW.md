---
pr: kubernetes-sigs/prow#783
title: "repoowners: add advisory_approvers OWNERS field"
head_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
base: main
reviewed_at: 2026-09-28T14:58:29Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-09-28T14:59:05Z
  gated_head_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
  reviewed_head_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
refresh_log:
  - at: 2026-09-28T14:58:29Z
    old_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
    new_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
    summary: "No code changes; incorporated the author's clarification on bot assignment and documentation."
---

## Gate

**Decision: do-not-merge.** The PR head is unchanged since the review. Three blocking findings still contradict the intended `advisory_approvers` behavior: advisory-only directories lose ordinary leaf suggestions, advisory-only root files fail validation, and blunderbuss can still request reviews from advisory approvers. The author confirmed on 2026-09-28 that bots should never assign or suggest them.

**Gating findings**

- **Blocks merge — `REVIEW.md`, advisory-only leaf:** `pkg/repoowners/repoowners.go:903-908` still stops at a leaf layer containing only advisory approvers, then subtracts them and returns no regular parent candidates. Fix the traversal/selection behavior or explicitly reject this configuration if unsupported.
- **Blocks merge — `REVIEW.md`, advisory-only root:** `pkg/plugins/verify-owners/verify-owners.go:459-466` still checks only `approvers` before adding `advisoryApprovers` to owners. Decide whether advisory-only root files are valid; then accept them or validate against them explicitly.
- **Blocks merge — `REVIEW.md`, blunderbuss fallback:** `pkg/plugins/blunderbuss/blunderbuss.go:118-123,338-344,421-426` still exposes all `Approvers()` as fallback reviewers when approver fallback is enabled and ordinary reviewers are insufficient. Filter advisory users through the entire fallback and cover this path in a test.
- **Must address before merge — `REVIEW.md` and @damdo, documentation:** `site/content/en/docs/components/plugins/approve/approvers/_index.md:28-42,70-82` still omits the field and its assignment semantics. Document the field and its interaction with `reviewers`, as the author agreed to do.
- **Must address before merge — `REVIEW.md`, maintainability:** `pkg/repoowners/repoowners.go:269-275,948-950` still does not document the dual-storage invariant or why `TopLevelApprovers` includes advisory users while `LeafApprovers` excludes them. Add the comments requested in the review.
- **Needs confirmation — `REVIEW.md`, advisory user also listed as reviewer:** `pkg/repoowners/repoowners_test.go` still expects such a user in `LeafReviewers`, which permits assignment as a reviewer. Confirm whether the author's stated never-assign intent includes this overlap and document the answer.

**Independent merge risk**

- `pkg/repoowners/repoowners.go:59` adds an optional exported config field and OWNERS key. Existing OWNERS files and defaults are unchanged; no existing deployment needs a migration. Repositories adopting the new key will encounter the selection and validation failures above, and mixed-version deployments may differ until every component understands the field. No flags, CRDs, or existing exported API signatures change.

## What this PR does

- Adds `advisory_approvers` to simple and filtered OWNERS configuration.
- Merges advisory users into `approvers` so they retain `/approve` authority and other approver privileges.
- Subtracts advisory users from `LeafApprovers`, removing them from approval-notifier suggestions.
- Extends verify-owners membership validation to advisory users.

Since previous review:
- No code changed. On 2026-09-28, smg247 [clarified](https://github.com/kubernetes-sigs/prow/pull/783#issuecomment-5871948669) that advisory approvers should never be assigned or suggested by the bot and agreed that the field needs documentation.

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

### [blocking] Advisory approvers remain eligible for blunderbuss fallback
- where: `pkg/plugins/blunderbuss/blunderbuss.go:118-120,421-426`
- concern: The fallback adapter exposes regular `Approvers()` as `Reviewers()`. After consuming leaf candidates, `getReviewers` fills the requested-reviewer count from that unfiltered set. Advisory approvers therefore can receive a review request whenever the regular leaf candidates are insufficient. The author [confirmed](https://github.com/kubernetes-sigs/prow/pull/783#issuecomment-5871948669) that they should never be assigned or suggested by the bot. Filter advisory users from the full fallback path and add coverage.
- excerpt: |
    func (foc fallbackReviewersClient) Reviewers(path string) layeredsets.String {
        return foc.ownersClient.Approvers(path)
    }

### [should-fix] Dual-storage invariant is under-documented
- where: `pkg/repoowners/repoowners.go:270` and `:756-769`
- concern: Advisory approvers live in both `o.advisoryApprovers` and `o.approvers`. Only documented by comment in LeafApprovers. Future methods reading `o.approvers` must know advisory users are mixed in. Add doc comment on the `advisoryApprovers` struct field. Flagged by code quality and maintainability reviewers independently.

### [should-fix] TopLevelApprovers/LeafApprovers asymmetry undocumented
- where: `pkg/repoowners/repoowners.go:948-950`
- concern: `TopLevelApprovers` includes advisory approvers (reads `o.approvers` directly), `LeafApprovers` subtracts them. Asymmetry is correct but undocumented. A future contributor might "fix" it. Add one-line comment.

### [should-fix] Document advisory_approvers for OWNERS authors
- where: `site/content/en/docs/components/plugins/approve/approvers/_index.md:28-42,70-82`
- concern: The OWNERS example and approver-selection explanation do not describe `advisory_approvers`. The author [agreed](https://github.com/kubernetes-sigs/prow/pull/783#issuecomment-5871948669) it should be documented. Explain that listed users retain approval rights but should not be assigned or suggested by bots, including the behavior when they also appear in `reviewers`.

### [question] Is advisory-only OWNERS a valid use case?
- where: both blocking findings
- concern: Both blocking bugs only trigger when an OWNERS file has `advisory_approvers` but no regular `approvers`. If this is unsupported, add explicit validation. If supported, fix both bugs.

### [question] Should advisory approvers listed as reviewers appear in LeafReviewers?
- where: `pkg/repoowners/repoowners_test.go` TestAdvisoryApproverAlsoReviewer
- concern: Test asserts advisory approver who is also a reviewer still appears in LeafReviewers (can be auto-assigned as reviewer, not as approver). Confirm intentional.

## Resolved

### [question] Is a distinct advisory_approvers field justified?
- where: `pkg/repoowners/repoowners.go:898-907` and PR discussion on 2026-09-23
- resolution: On 2026-09-28, smg247 [said](https://github.com/kubernetes-sigs/prow/pull/783#issuecomment-5871948669) the field is worth merging. The documentation work is tracked separately above.

## Checked
- RepoOwner no longer exposes AdvisoryApprovers; the six fake-only method implementations were removed
- Approval-notifier candidate selection uses LeafApprovers (excluding advisory is correct)
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
