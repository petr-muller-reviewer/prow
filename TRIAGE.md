---
issue: kubernetes-sigs/prow#960
title: "Race condition applying \"area\" labels"
state: open
labels: []
main_sha: f56daaa3a55d245be5a03acd0dea5960a62db304
triaged_at: 2026-09-22T22:08:58Z
verdict: accepted
---

## Findings

### [reproducibility] Linked Autoscaler PR retains contradictory labels
- detail: `kubernetes/autoscaler#10326` has both `area/vertical-pod-autoscaler` and `do-not-merge/needs-area`.
- evidence: Its GitHub timeline records Prow adding the area label at `2026-09-21T10:52:19Z` and the blocker at `2026-09-21T10:52:20Z`.

### [cause] Concurrent handlers use a non-atomic label snapshot
- detail: `owners-label` and `require-matching-label` handle the PR-open event concurrently. The latter waits its grace period, reads labels, then adds its blocker; the former can add an area label after that read.
- evidence: `pkg/hook/events.go:174-191`; `pkg/plugins/require-matching-label/require-matching-label.go:204-247`.

### [related-code] Hook dispatch has no ordering between plugins
- where: `pkg/hook/events.go:174-191`
- excerpt: |
    for p, h := range s.Plugins.PullRequestHandlers(...) {
        s.wg.Add(1)
        go func(p string, h plugins.PullRequestHandler) { ... }(p, h)
    }

### [related-code] OWNERS labels are applied after file and label reads
- where: `pkg/plugins/owners-label/owners-label.go:71-119`
- excerpt: |
    changes, err := ghc.GetPullRequestChanges(org, repo, number)
    ...
    for _, labelToAdd := range sets.List(neededLabels.Difference(currentLabels)) {
        if err := ghc.AddLabel(org, repo, number, labelToAdd); err != nil { ... }
    }

### [related-code] Matching-label grace period is only a heuristic
- where: `pkg/plugins/config.go:984-988,1349-1352`
- excerpt: |
    // GracePeriod is the amount of time to wait ... before ... labels.
    // Defaults to '5s'.
    if rml.GracePeriod == "" {
        c.RequireMatchingLabel[i].GracePeriod = "5s"
    }

### [related-code] Reconciliation requires another handler invocation
- where: `pkg/plugins/require-matching-label/require-matching-label.go:235-247`
- excerpt: |
    if hasMatchingLabel && hasMissingLabel {
        ghc.RemoveLabel(e.org, e.repo, e.number, cfg.MissingLabel)
    } else if !hasMatchingLabel && !hasMissingLabel {
        ghc.AddLabel(e.org, e.repo, e.number, cfg.MissingLabel)
    }

### [related-pr] Concrete reproduction
- ref: kubernetes/autoscaler#10326
- relevance: VPA files map to `vertical-pod-autoscaler/OWNERS`, which declares `area/vertical-pod-autoscaler`, but the configured rule still added the missing-area blocker.

## Checked
- Issue #960 is open, unlabelled, has no comments, and is in Prow scope.
- The linked PR's public label timeline and its OWNERS mapping.
- `kubernetes/test-infra` requires `^area/` on `kubernetes/autoscaler` PRs and uses `do-not-merge/needs-area` as the missing label.
- Existing owners-label and require-matching-label tests cover static decisions, not this interleaving or eventual convergence.
- No matching earlier Prow issue was found.

## Next steps
- Apply `kind/bug`, `area/hook`, and optionally `help-wanted`.
- Add bounded eventual reconciliation to `require-matching-label` after it adds a missing-label blocker; remove the blocker/comment when a matching label subsequently appears.
- Add a deterministic regression test for a late OWNERS-derived area label and the converged final state.
- Consider a longer Autoscaler `grace_period` in `kubernetes/test-infra` as mitigation; it cannot guarantee correctness.

## Open questions
- Should reconciliation apply to every require-matching-label rule or be opt-in?
- What bounded recheck policy prevents label/comment thrashing?
