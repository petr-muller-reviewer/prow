---
issue: kubernetes-sigs/prow#784
title: "Add advisory_approvers field to OWNERS files"
state: open
labels:
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:29:48Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/feature, area/repoowners]
---

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

- Confirm with the author/maintainers whether the field name `advisory_approvers` is final, or whether alternatives (e.g. `silent_approvers`, `non_assignable_approvers`) should be considered before implementation starts.
- Point the author at the `RequiredReviewers` code paths (`repoowners.go:58,270,489,746-750,909-915`) as the structural template: new `Config` field → new `o.advisoryApprovers` map → wiring in `applyConfigToPath` → union into `Approvers`/`AllApprovers` → leave `LeafApprovers` untouched.
- Apply `kind/feature` and `area/repoowners` labels.

## Open questions

- Should advisory approvers be visible in any tooling/UI that lists approvers (e.g. `AllApprovers`), or only recognized for `/approve` authorization specifically?
- Any interaction expected with `no_parent_owners` or nested-OWNERS layering beyond the default "same as `approvers`" behavior?

## Briefing Completed

Briefed maintainer on: 2026-07-27

Key questions asked: none — maintainer confirmed through all 7 slides without requesting elaboration.

Maintainer decision: Keep issue open, apply `kind/feature` / `area/repoowners` labels, let the already-self-assigned author (smg247) proceed with the two-map (`o.approvers` / `o.advisoryApprovers`) design; confirm the `advisory_approvers` field name with the author before implementation locks in.
