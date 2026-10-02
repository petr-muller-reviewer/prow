---
pr: kubernetes-sigs/prow#783
title: "repoowners: add advisory_approvers OWNERS field"
head_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
base: main
reviewed_at: 2026-10-01T15:51:54Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-10-01T15:52:31Z
  gated_head_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
  reviewed_head_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
refresh_log:
  - at: 2026-09-28T14:58:29Z
    old_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
    new_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
    summary: "No code changes; incorporated the author's clarification on bot assignment and documentation."
  - at: 2026-10-01T15:51:54Z
    old_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
    new_sha: 682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57
    summary: "No code changes; recorded Prucek's approval, the approval-notifier update, and the PR merge."
---

## Gate

**Decision: do-not-merge (historical; already merged).** GitHub merged the unchanged head `682e7cd91945551b4f5b64a7bbb73b1f1b1ecc57` at 2026-10-01T13:03:55Z. The substantive decision remains do-not-merge because the current code still conflicts with the author's stated guarantee that bots never assign or suggest advisory approvers. This record cannot prevent the completed merge.

**Unaddressed gating findings**

- **`REVIEW.md`, advisory-only leaf — blocks merge:** `pkg/repoowners/repoowners.go:903-908` stops at a leaf layer containing only advisory approvers, subtracts them, and returns no regular parent candidates. Fix traversal/selection or explicitly reject the configuration.
- **`REVIEW.md`, advisory-only root — blocks merge:** `pkg/plugins/verify-owners/verify-owners.go:459-466` checks only `approvers` before adding `advisoryApprovers` to owners. Decide whether advisory-only root files are valid, then accept or explicitly reject them.
- **`REVIEW.md`, blunderbuss fallback — blocks merge:** `pkg/plugins/blunderbuss/blunderbuss.go:118-123,421-426` exposes all `Approvers()` as fallback reviewers after leaf candidates are exhausted. Filter advisory users through that path and add coverage.
- **`REVIEW.md` and @damdo, documentation — should resolve:** `site/content/en/docs/components/plugins/approve/approvers/_index.md:28-42,70-82` still omits the field and its assignment semantics, despite the author's agreement to document it.

**Independent merge risk**

- `pkg/repoowners/repoowners.go:59` adds an optional OWNERS configuration field, so existing configurations need no migration. Every repository that adopts the key can encounter the selection and validation failures above; mixed-version deployments can interpret it differently until every component understands the field. No flags, CRDs, or existing exported API signatures change.

## What this PR does

- Adds `advisory_approvers` to simple and filtered OWNERS configuration.
- Merges advisory users into `approvers` so they retain `/approve` authority and other approver privileges.
- Subtracts advisory users from `LeafApprovers`, removing them from approval-notifier suggestions.
- Extends verify-owners membership validation to advisory users.

Since previous review:
- No code changed. On 2026-09-28, smg247 [clarified](https://github.com/kubernetes-sigs/prow/pull/783#issuecomment-5871948669) that advisory approvers should never be assigned or suggested by the bot and agreed that the field needs documentation.
- Prucek approved the unchanged head at 2026-10-01T12:41:53Z; approval-notifier reported the PR approved at 12:42:01Z, and GitHub merged it at 13:03:55Z.

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

## Followups

### Document advisory_approvers for OWNERS authors

- category: docs
- necessity: should
- source: post-merge followup to PR #783 "repoowners: add advisory_approvers OWNERS field" (merge commit `42422d8f2142fd480bdc7c624c8195fe58c10130`)

```text
In kubernetes-sigs/prow, following merged PR #783 "repoowners: add advisory_approvers OWNERS field" (merge commit 42422d8f2142fd480bdc7c624c8195fe58c10130), document advisory_approvers in site/content/en/docs/components/plugins/approve/approvers/_index.md.

Add a concise OWNERS YAML example and explain the field's intended relationship to approvers and reviewers: advisory users retain approval authority, should be listed in advisory_approvers rather than duplicated under approvers, and selection behavior differs from ordinary approvers. Explain the approval-notifier and review-assignment implications in terms verified by the current code.

Before asserting that advisory users are never assigned or suggested, reconcile that wording with the current blunderbuss fallback behavior in pkg/plugins/blunderbuss/blunderbuss.go:118-123,421-426. Do not publish an unconditional guarantee that the implementation does not meet; document the actual behavior or explicitly leave that claim for the separate bug-fix followup.

Acceptance criteria:
- The public approvers guide includes advisory_approvers in its OWNERS example or an adjacent focused example.
- It explains approval authority, the intended non-duplication with approvers, and interaction with reviewers.
- It accurately distinguishes approval-notifier suggestions from blunderbuss review requests, without overstating current guarantees.

Out of scope: changing repoowners selection logic, blunderbuss fallback behavior, verify-owners validation, or unrelated OWNERS documentation.
```

### Restore advisory non-assignment invariant

- category: post-merge bug fix
- necessity: must
- source: post-merge followup to PR #783 "repoowners: add advisory_approvers OWNERS field" (merge commit `42422d8f2142fd480bdc7c624c8195fe58c10130`)

```text
In kubernetes-sigs/prow, following merged PR #783 "repoowners: add advisory_approvers OWNERS field" (merge commit 42422d8f2142fd480bdc7c624c8195fe58c10130), restore the invariant that advisory approvers retain approval authority but are never suggested or assigned by Prow bots.

Fix the advisory-only leaf case in pkg/repoowners/repoowners.go:898-907 so a nested OWNERS file containing only advisory_approvers does not suppress ordinary parent candidates after advisory users are excluded. Also fix the blunderbuss approver-fallback path in pkg/plugins/blunderbuss/blunderbuss.go:118-123,421-426 so it cannot select advisory users after ordinary leaf candidates are exhausted.

Add focused regression coverage in the existing repoowners, approve/approvers, and blunderbuss test suites as appropriate. Cover both approval-notifier candidate/suggestion behavior and blunderbuss review-request behavior; retain advisory users' /approve authority and do not change ordinary approver or reviewer selection semantics.

Acceptance criteria:
- An advisory-only child OWNERS file does not cause LeafApprovers to return an empty ordinary-candidate set when regular inherited approvers exist.
- Advisory users never appear in approval-notifier suggested approvers or in blunderbuss fallback review requests.
- Advisory users remain valid approvers through the normal authorization path.
- Regression tests fail on the merged implementation and pass with the fix.

Out of scope: public documentation changes, unrelated OWNERS selection refactors, and changing the meaning of ordinary approvers or reviewers.
```

### Define and enforce advisory-only OWNERS validity

- category: configuration semantics
- necessity: must
- source: post-merge followup to PR #783 "repoowners: add advisory_approvers OWNERS field" (merge commit `42422d8f2142fd480bdc7c624c8195fe58c10130`)

```text
In kubernetes-sigs/prow, following merged PR #783 "repoowners: add advisory_approvers OWNERS field" (merge commit 42422d8f2142fd480bdc7c624c8195fe58c10130), define and consistently enforce whether an OWNERS file containing only advisory_approvers is valid.

Investigate pkg/plugins/verify-owners/verify-owners.go:459-466 and the corresponding repoowners loading/selection behavior. If advisory-only OWNERS files are supported, update root validation to accept them and ensure nested inheritance is correct. If they are unsupported, reject them at every level with a clear, consistent validation error before they can produce empty leaf candidates.

Add root and nested OWNERS regression tests that make the chosen rule explicit, including filtered/full configuration if the field supports those forms.

Acceptance criteria:
- Root and nested advisory-only OWNERS files have one documented, consistently enforced validity rule.
- verify-owners and runtime repoowners behavior agree on that rule.
- Tests cover the root case, a nested inherited-approver case, and the rejected case if the project chooses rejection.

Out of scope: changing advisory assignment behavior beyond what is necessary to enforce the selected validity rule, rewriting unrelated OWNERS validation, or adding public documentation beyond a small clarification required by the chosen rule.
```
