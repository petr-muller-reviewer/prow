---
issue: kubernetes-sigs/prow#784
title: "Add advisory_approvers field to OWNERS files"
state: open
labels:
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-10-10T16:24:59Z
verdict: resolved
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/feature, area/repoowners]
refresh_log:
  - at: 2026-10-10T16:24:59Z
    since: 2026-07-27T00:29:48Z
    summary: "Recorded PR #783 as a sufficient resolution; it merged on 2026-10-01."
advice:
  advised_at: 2026-10-10T16:25:32Z
  based_on_triaged_at: 2026-10-10T16:24:59Z
---

# Triage

## Verdict

Resolved. PR #783 implements the requested advisory approver behavior and merged on 2026-10-01. Issue #784 remains open on GitHub, with no labels.

## Resolution

[kubernetes-sigs/prow#783](https://github.com/kubernetes-sigs/prow/pull/783), “repoowners: add advisory_approvers OWNERS field,” merged on 2026-10-01. Classification: **sufficient**.

- `pkg/repoowners/repoowners.go` adds `AdvisoryApprovers` and a separate `advisoryApprovers` map, adds advisory users to the full approver set, and removes them from `LeafApprovers()`.
- Tests cover advisory approvers in simple and filtered OWNERS configs, `/approve` recognition, exclusion from auto-assignment, and continued reviewer eligibility when also listed as a reviewer.
- `pkg/plugins/verify-owners/verify-owners.go` includes advisory approvers in OWNERS validation.

This matches the prior recommendation to use a separate advisory map, include it in full approver authorization, and keep it out of leaf-only assignment.

## What the issue reports

- Add `advisory_approvers` to OWNERS so listed maintainers keep `/approve` authority without being automatically assigned; if also listed as reviewers, they remain eligible for reviewer selection.

Since previous triage:

- PR #783, authored by smg247 and cross-referenced before the prior triage, merged on 2026-10-01 with the implementation described above.
- Issue #784 remains open and unlabeled. There are no issue comments after the prior triage timestamp and no PR comments or reviews after the merge.

## Findings

### [related-code] Config struct declares approver/reviewer fields
- where: `pkg/repoowners/repoowners.go:52-59`
- excerpt: |
    type Config struct {
        Approvers         []string `json:"approvers,omitempty"`
        Reviewers         []string `json:"reviewers,omitempty"`
        RequiredReviewers []string `json:"required_reviewers,omitempty"`
        Labels            []string `json:"labels,omitempty"`
    }

### [related-code] applyConfigToPath copies config into per-path maps
- where: `pkg/repoowners/repoowners.go:731-753`
- excerpt: |
    if len(config.Approvers) > 0 {
        if o.approvers[path] == nil {
            o.approvers[path] = make(map[*regexp.Regexp]sets.Set[string])
        }
        o.approvers[path][re] = o.ExpandAliases(NormLogins(config.Approvers))
    }
    ...
    if len(config.RequiredReviewers) > 0 {
        if o.requiredReviewers[path] == nil {
            o.requiredReviewers[path] = make(map[*regexp.Regexp]sets.Set[string])
        }
        o.requiredReviewers[path][re] = o.ExpandAliases(NormLogins(config.RequiredReviewers))
    }

### [related-code] entriesForFile: leaf vs. full accessors share one map
- where: `pkg/repoowners/repoowners.go:844-916`
- detail: `entriesForFile(path, people, leafOnly)` walks from a file's directory to root. `leafOnly=true` stops at the first directory with a match (`LeafApprovers`, `LeafReviewers`); `leafOnly=false` accumulates the whole path (`Approvers`, `Reviewers`, `AllApprovers`, `AllReviewers`). `LeafApprovers` and `Approvers` both read `o.approvers` — the same map — so a user can't be in the "full" set without also being in the "leaf" set today. This is why advisory approvers need a second, parallel map rather than a new entry in the existing one.

### [related-code] /approve authorization consumes the "full" Approvers set
- where: `pkg/plugins/approve/approvers/owners.go:78,476`
- detail: `repo.Approvers(toApprove).Set()` is used both to compute available approvers and to check if current approvers satisfy the requirement — this is the layered (non-leaf) accessor, so advisory approvers must be included here.

### [related-code] Blunderbuss auto-assignment consumes the leaf-only accessors
- where: `pkg/plugins/blunderbuss/blunderbuss.go:100-123`
- detail: `LeafReviewers`/`LeafApprovers` (via `fallbackReviewersClient.LeafReviewers` which itself calls `LeafApprovers`) drive auto-assignment suggestions. Advisory approvers must be excluded here — the two-map design achieves this structurally, without new filtering logic at this call site.

### [related-code] RequiredReviewers is the direct structural precedent
- where: `pkg/repoowners/repoowners.go:58,270,489,746-750,909-915`
- detail: `RequiredReviewers` was added as a fully independent field + map + `applyConfigToPath` block + accessor, the exact shape needed for `AdvisoryApprovers`. A contributor can follow this end-to-end as a template.

### [related-issue] kubernetes-sigs/prow#780
- ref: kubernetes-sigs/prow#780
- relevance: "Smart blunderbuss reviewer selection using git blame data" — same author (smg247), #784 was explicitly split out of it.

## Checked

- Searched for any existing `emeritus_approvers`-style field in this repo (`grep -ri emeritus`) — only unrelated prose in `pkg/plugins/reward-owners/reward-owners.go`; no dead/existing implementation to build on or conflict with.
- Confirmed `/approve` authorization and blunderbuss auto-assignment read from different accessor methods (`Approvers` vs. `LeafApprovers`) that are nonetheless backed by the *same* underlying map today — the crux of why a naive single-field addition can't satisfy both "recognized by /approve" and "never auto-assigned" simultaneously.
- Checked for an OWNERS JSON schema or generated docs needing updates — none exist; `required_reviewers` (closest analog) isn't documented in any schema file either, only in code comments, so no schema-regeneration step is required.
- Confirmed the request is well-specified (exact semantics, example OWNERS snippet, clear motivation) and already self-assigned by the author.

## Next steps

- No implementation work remains; PR #783 supplies the requested behavior. The issue is still open and unlabeled, so a maintainer can close it and apply the suggested labels if that is the project's process.

## Advice

- **Close #784 as resolved by PR #783.** The PR merged on 2026-10-01 and issue #784 is still open. No labels or comments have changed since the refreshed triage.

  ```bash
  gh issue close 784 --repo kubernetes-sigs/prow --comment $'> *This was generated by AI during triage.*\n\nResolved by #783, merged 2026-10-01. It adds `advisory_approvers` with `/approve` authority while excluding advisory approvers from `LeafApprovers()` auto-assignment.'
  ```

## Open questions

- Should advisory approvers be visible in any tooling/UI that lists approvers (e.g. `AllApprovers`), or only recognized for `/approve` authorization specifically?
- Any interaction expected with `no_parent_owners` or nested-OWNERS layering beyond the default "same as `approvers`" behavior?

## Briefing Completed

Briefed maintainer on: 2026-07-27

Key questions asked: none — maintainer confirmed through all 7 slides without requesting elaboration.

Maintainer decision: Keep issue open, apply `kind/feature` / `area/repoowners` labels, let the already-self-assigned author (smg247) proceed with the two-map (`o.approvers` / `o.advisoryApprovers`) design; confirm the `advisory_approvers` field name with the author before implementation locks in.
