---
issue: kubernetes-sigs/prow#610
title: "owners-label should ignore merge commits"
state: closed
labels: kind/feature, lifecycle/rotten
main_sha: ea9e703d576df6d125159beb54d30d90ba7e8634
triaged_at: 2026-07-27T00:32:03Z
verdict: accepted
refresh_log:
  - at: 2026-05-03
    summary: Initial triage; good-first-issue, kind/feature labels; 6 comments, consensus on unconditional merge-commit skip
  - at: 2026-06-08T17:37:43Z
    summary: lifecycle/stale applied 2026-05-07, lifecycle/rotten applied 2026-06-06 by k8s-triage-robot; good-first-issue label removed; auto-close risk in ~30d
  - at: 2026-07-27T00:32:03Z
    summary: Full re-triage. Issue auto-closed "not planned" by k8s-triage-robot on 2026-07-06 (inactivity, not rejection). Discovered PR kubernetes-sigs/prow#782 (open, CI green, mergeable) already implements the fix as opt-in per-repo config, not the unconditional skip the issue thread converged on. Maintainer briefed; agreed to reopen #610 and steer review of #782.
---

## Findings

### [cause] `owners-label` has no per-commit visibility and never removes labels
- detail: `GetPullRequestChanges` returns the full diff between PR head and base (GitHub `GET /pulls/{n}/files`), with no per-commit breakdown. When a merge commit is present, every file from the merged branch appears in this diff. `owners-label`'s `githubClient` interface has no `RemoveLabel` method at all, so erroneous labels persist after the merge commit is force-pushed away.
- evidence: `pkg/plugins/owners-label/owners-label.go:77-84` (file fetching/label mapping), `pkg/plugins/owners-label/owners-label.go:108-117` (add-only label application)

### [related-code] DCO plugin precedent for merge-commit detection
- where: `pkg/plugins/dco/dco.go:156-160`
- excerpt: |
    isMerge := len(commit.Parents) > 1
    if !isMerge && !testRe.MatchString(commit.Commit.Message) {
        commitsMissingDCO = append(commitsMissingDCO, commit)
    }
- relevance: Establishes the `len(commit.Parents) > 1` pattern for detecting merge commits via `ListPullRequestCommits` (`pkg/github/client.go:4810`), which returns `Parents` but not `Files` (only `GetSingleCommit`, `pkg/github/client.go:2789`, populates `Files`/`Stats` per `pkg/github/types.go:1354-1369`). Confirmed unchanged since prior triage.

### [related-pr] PR #782 implements the fix, but as opt-in rather than unconditional
- ref: kubernetes-sigs/prow#782
- relevance: Open PR by carterpewpew, `Fixes: #610`, adds `Owners.IgnoreMergeCommits []string` config (`pkg/plugins/config.go`) and skips labeling in `handle()` when any commit has `len(Parents) > 1` and the repo opted in (`pkg/plugins/owners-label/owners-label.go`). CI fully green (unit, race-detector, integration, lint, image-build), mergeable, but not yet lgtm/approved — one `COMMENTED` review from `Prucek` (2026-07-02), and the triaging maintainer already ran `/ok-to-test` (2026-06-30). Deviates from the issue thread's consensus that the skip should be unconditional (see below) — implements BenTheElder's earlier opt-in framing instead. +107/-3 across 3 files; includes `TestHandleIgnoreMergeCommits` covering both merge and non-merge cases.

### [reproducibility] Original repro is unchanged and unaffected by anything since
- detail: Push a merge commit onto a PR branch → `owners-label` fires on `synchronize` and labels every file touched by the merge → force-pushing the merge commit away does not remove the labels, since the plugin never removes labels.
- evidence: `pkg/plugins/owners-label/owners-label.go:58-61` (event filter: opened/reopened/synchronize)

## Checked

- Re-verified all code-path claims from the prior triage against current `main` (`ea9e703`) — line numbers shifted slightly, behavior unchanged.
- Confirmed no partial/alternate implementation of the fix landed on `main` since the last triage (`git log` on `owners-label.go` shows zero commits since the pkg/cmd/test repo reorg; `grep -ni "merge\|parent"` on the file is empty).
- Searched the issue's GitHub timeline for cross-references — found PR #782 via a `cross-referenced` event.
- Confirmed PR #782's CI status (all jobs green) and review/approval state (commented only, not approved, not lgtm'd; tide reports "Not mergeable. Needs approved, lgtm labels.").
- Briefed the maintainer through all 7 slides; no questions raised, no objection to the recommended plan.

## Next steps

- Reopen #610 (`/reopen`) — closed by bot inactivity timeout, not resolved or rejected.
- Remove `lifecycle/rotten` (`/remove-lifecycle rotten`) once reopened, to reset the stale-bot clock while #782 is under review.
- Review PR #782 on its technical merits; explicitly raise the opt-in-vs-unconditional discrepancy against the issue thread's consensus before approving.
- Once #782 merges, close #610 referencing the PR (not "not planned").

## Open questions

- Should `ignore_merge_commits` ship opt-in (as in #782) or should reviewers push for the unconditional behavior the issue thread agreed was correct?
- Is there a `mergecommitblocker`-using repo that should be defaulted into `ignore_merge_commits` at rollout, to avoid a second follow-up PR?
