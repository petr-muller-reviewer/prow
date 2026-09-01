---
issue: kubernetes-sigs/prow#912
title: "tide fails to merge stacked PRs that use github's new stacked PR system"
state: open
labels:
main_sha: 88f56c0e89c868ec4e6bf0305fe23c7efa984ae7
triaged_at: 2026-08-31T18:04:33Z
verdict: accepted
---

## Findings

### [reproducibility] Classic merge endpoint rejects stacked PRs
- detail: When Tide tries to merge a PR that is part of GitHub's new stacked-PR system, GitHub returns "Merging stacked PRs via this endpoint is not supported. Use the asynchronous merge endpoint instead." Reported directly by the issue author; no repro steps needed beyond having a stacked PR in a Tide-managed repo.
- evidence: issue body, kubernetes-sigs/prow#912.

### [cause] Discovery misses non-bottom stack members
- detail: Tide's GitHub search query is built per-org from `IncludedBranches`/`ExcludedBranches`, emitted as `base:"<branch>"` / `-base:"<branch>"` qualifiers. Only the bottom PR of a stack has `base == main`/`master` (or another configured target branch); every PR above it targets a synthetic intermediate branch that never matches the query, so it's invisible to Tide, not merely mis-pooled.
- evidence: `pkg/config/tide.go:596-644` (`TideQuery.constructQuery`), `pkg/config/tide.go:657` (`OrgQueries`).

### [cause] Subpool partitioning splits stack members across pools
- detail: PRs are grouped into subpools keyed by `org/repo:branch`. Even if all stack members were discovered, each would land in a different subpool because each targets a different base branch, so Tide's status/blocker/merge logic (which operates independently per subpool) can never evaluate a stack as one unit.
- evidence: `pkg/tide/tide.go:1902-1920` (`subpool` struct), `:1934-1936` (`poolKey`), `:1940-1967` (`dividePool`).

### [cause] Merge call assumes one API call resolves exactly one PR, synchronously
- detail: `mergePRs` iterates PRs one at a time and calls the classic synchronous merge endpoint per PR, sleeping 5s between calls to let GitHub "recalculate mergeability." GitHub's new async merge endpoint for stacked PRs (`PUT /pulls/{n}/merge-async`) instead merges the whole stack up to the given PR in a single call and requires polling a job UUID for the result — a fundamentally different call/response shape that Tide has no support for.
- evidence: |
    `pkg/tide/github.go:243-297`:
    ```go
    for i, pr := range prs {
        ...
        keepTrying, err := tryMerge(func() error {
            ghMergeDetails := gi.prepareMergeDetails(commitTemplates, pr, *mergeMethod)
            return gi.ghc.Merge(sp.org, sp.repo, pr.Number, ghMergeDetails)
        })
        ...
        if err == nil && i+1 < len(prs) {
            sleep(time.Second * 5)
        }
    }
    ```
    `pkg/github/client.go:4236-4247` (`client.Merge`): single synchronous `PUT /repos/{org}/{repo}/pulls/{pr}/merge`, no async/job-polling variant exists.

### [related-code] No existing stack-awareness anywhere in the codebase
- where: repo-wide (excluding vendor)
- excerpt: |
    Search for `stacked`, `merge-async`, `merge_async`, `MergeAsync`, `MergeQueue` returns nothing relevant — one unrelated comment in `pkg/statusreconciler/doc.go:18` about GitHub's older native merge-queue feature. This is a greenfield gap: no `MergeAsync` client method, no job/UUID polling helper, no cross-subpool grouping concept.

### [related-code] Test extension points
- where: `pkg/tide/tide_test.go:933` (`TestDividePool`), `pkg/tide/tide_test.go:2689` (`TestFilterSubpool`), `pkg/tide/github_test.go:182` (`TestPrepareMergeDetails`)
- excerpt: |
    Natural homes for stack-grouping regression tests. None of the existing merge tests assert on the sequence/count of `Merge()` calls for a subpool — that coverage would need to be added for any multi-PR-per-call semantics.

## Checked
- Repo-wide grep for `stacked`, `merge-async`, `merge_async`, `MergeAsync`, `MergeQueue` — no existing scaffolding.
- Confirmed Tide (`pkg/tide/`, `cmd/tide/`) is in-scope for this repository.
- Read the maintainer (Prucek)'s scoping comment on the issue, which already breaks the problem into the same three root causes independently confirmed above.

## Next steps
- Confirm whether a maintainer or contributor will own a design doc covering discovery, pooling, and async-merge semantics before implementation starts (maintainer explicitly requested this).
- Verify stability/maturity of GitHub's stacked-PR async merge API before committing to a design.
- If incremental progress is desired, scope and label a narrower stopgap separately (swap in the async merge endpoint only for detected stack members, without fixing discovery/pooling) and mark it explicitly as partial.
- Apply labels `area/tide`, `kind/feature`; hold off on `good-first-issue`/`help-wanted` until a design exists.

## Open questions
- Does GitHub's stacked-PR async merge API have documented stability/versioning guarantees suitable to build against?
- Should stack-awareness be opt-in per org/repo via config, or auto-detected?
- Is a design doc already in progress, or does one need to be initiated from scratch?
