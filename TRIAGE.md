---
issue: kubernetes-sigs/prow#937
title: "tide: PR excluded for unfetchable changed files gets no status and is retried forever"
state: open
labels: kind/bug, area/tide
main_sha: d91e0f84e0f2050b41390b28b69a05f5284ba4a2
triaged_at: 2026-09-15T21:30:56Z
verdict: accepted
---

## Findings

### [reproducibility] A changed-files 422 repeatedly excludes the PR
- detail: A PR with a file-dependent presubmit is excluded from its subpool on each Tide sync when GitHub cannot generate its changed-file list. PR #930 deliberately allows unaffected peers to continue, but the excluded PR is retried indefinitely without an actionable explanation.
- evidence: kubernetes-sigs/prow#937; kubernetes-sigs/prow#914.

### [cause] Exclusion discards the only diagnostic
- where: `pkg/tide/tide.go:1714-1729`
- excerpt: |
    shouldRun, err := ps.ShouldRun(sp.branch, c.changedFiles.prChanges(&pr), forceRun, false)
    if err != nil {
        changedFilesErr = err
        break
    }
    if changedFilesErr != nil {
        log.WithError(changedFilesErr).Warn("Failed to determine required presubmits for PR, excluding from subpool")
        delete(presubmits, pr.Number)
        continue
    }
- relevance: The PR is removed from `sp.prs`; no structured error survives for either controller.

### [cause] Status evaluation mistakes exclusion for ordinary non-membership
- where: `pkg/tide/status.go:300-362`
- excerpt: |
    if _, ok := pool[prKey(crc)]; !ok {
        ...
        if !hasFulfilledQuery {
            return github.StatusPending, fmt.Sprintf(statusNotInPool, minDiff), nil
        }
    }
    ...
    return github.StatusSuccess, poolStatus(pr, mergeBlocksPolicy, log), nil
- relevance: An otherwise query-matching excluded PR falls through to ProwJob lookup and can receive generic retesting or `In merge pool.` status.

### [related-code] The sync/status handoff has no diagnostic field
- where: `pkg/tide/tide.go:579-585`; `pkg/tide/status.go:97-110`
- detail: `sync` transfers only blockers, pooled PRs, base SHAs, and required contexts into `statusUpdate`. Add a PR-keyed exclusion-reason map independent of `poolPRMap(filteredPools)`.

### [related-code] Existing Pool output cannot show an empty excluded subpool
- where: `pkg/tide/tide.go:733-745,1790-1871`
- detail: `filterSubpool` returns nil when `sp.prs` is empty, and `syncSubpool` assigns `Pool.Error` only from action-time errors. A diagnostic-only Pool representation is needed to satisfy the dashboard part of the report when every PR was excluded.

### [related-code] Focused test patterns already exist
- where: `pkg/tide/tide_test.go:3641-3775`; `pkg/tide/status_test.go:~70-875`
- detail: #930's test intercepts a 422 for one PR and proves that a peer remains eligible. The status table tests `expectedStatus` state and description; extend both for diagnostic retention and precedence.

### [related-pr] #930 intentionally left status reporting for follow-up
- ref: kubernetes-sigs/prow#930
- relevance: Merged commit `0d14409f7` changes changed-files failures from subpool-wide failure to per-PR exclusion. Review discussion explicitly agreed to surface the individual failure in a separate PR.

### [related-issue] #914 describes the underlying external limitation
- ref: kubernetes-sigs/prow#914
- relevance: GitHub returns 422 when a very large PR's changed-file diff is unavailable; retries cannot resolve this inside Tide.

### [related-issue] #777 has the same silent-subpool diagnostic class
- ref: kubernetes-sigs/prow#777
- relevance: A context-checker error can drop a whole subpool without a status. Keep #937 narrowly scoped, but design diagnostic plumbing so it can be reused later.

## Checked
- Issue #937 remains open with `kind/bug` and `area/tide`; no new activity since its creation.
- Checkout is `937-triage` at `d91e0f84e0f2050b41390b28b69a05f5284ba4a2`, containing merged #930.
- Traced `presubmitsByPull`, `filterSubpools`, `statusUpdate`, `expectedStatus`, `syncSubpool`, Pool construction, and the focused tests.
- Read #914, #777, and #930's maintainer discussion; none already resolves status propagation.

## Next steps
- Accept as a moderate Tide bug; retain `kind/bug` and `area/tide`, optionally add `help-wanted`.
- Preserve a PR-keyed changed-files exclusion reason through filtering and `statusUpdate`; return `github.StatusError` before ordinary pool/query logic.
- Add tests for one excluded PR with a surviving peer and for all PRs excluded.
- Decide whether a fully excluded subpool remains as a diagnostic-only Pool; this is required for dashboard visibility.

## Open questions
- Should `GetPresubmits` per-PR errors use the same user-facing diagnostic path?
- Should a fully excluded subpool be retained as a diagnostic-only Pool, or should this change guarantee only PR status?
- Should the plumbing be generalized for #777 now, or remain limited to changed-files exclusion?
