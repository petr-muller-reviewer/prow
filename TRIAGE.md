---
issue: 610
title: "owners-label should ignore merge commits"
repo: kubernetes-sigs/prow
reporter: danwinship
reported_at: 2026-02-05
triaged_at: 2026-06-08T17:37:43Z
verdict: LEGITIMATE
effort: Level 1
labels:
  - kind/feature
  - lifecycle/rotten
refresh_log:
  - at: 2026-05-03
    summary: Initial triage; good-first-issue, kind/feature labels; 6 comments, consensus on unconditional merge-commit skip
  - at: 2026-06-08T17:37:43Z
    summary: lifecycle/stale applied 2026-05-07, lifecycle/rotten applied 2026-06-06 by k8s-triage-robot; good-first-issue label removed; auto-close risk in ~30d
---

# Issue #610: `owners-label` should ignore merge commits

**LEGITIMATE** | **Level 1** | **kind/feature** | **lifecycle/rotten**
Reported by **danwinship** · 2026-02-05 · [View on GitHub](https://github.com/kubernetes-sigs/prow/issues/610)

## Quick Reference

| | |
|---|---|
| **Problem** | Merge commits cause flood of irrelevant labels |
| **Root Cause** | `GetPullRequestChanges` returns aggregate diff |
| **Labels stick** | Plugin only adds, never removes |
| **Fix Approach** | Skip labeling when merge commits detected |
| **Pattern** | DCO plugin (`dco.go:156-161`) |
| **Scope** | ~15-25 LOC in 1 file + tests |

## Since Previous Triage (2026-05-03)

- **2026-05-07**: `k8s-triage-robot` applied `lifecycle/stale` due to 90d inactivity; `good-first-issue` label removed in the same sweep.
- **2026-06-06**: `k8s-triage-robot` applied `lifecycle/rotten`. Issue will be **auto-closed in ~30 days** (around 2026-07-06) if no activity.
- No new substantive comments or linked PRs. Analysis and recommended fix unchanged.

## Discussion Summary

6 comments between **danwinship** (reporter) and **BenTheElder** (maintainer). Key points:

**Consensus reached:** Merge commits should *always* be skipped for labeling, regardless of whether `mergecommitblocker` is enabled. There is no workflow where labeling from merge commits makes sense.

- **BenTheElder**: +1, initially suggested opt-in tied to `mergecommitblocker`
- **danwinship**: Argued the fix should be unconditional — labeling from merges only produces noise
- **danwinship** proposed per-commit pseudocode: skip commits with `len(Parents) > 1`
- **Key insight**: Merging master into a PR would label every file changed upstream since branch point — never useful

## Root Cause

`owners-label` calls `GetPullRequestChanges` which hits GitHub's `GET /pulls/{n}/files` endpoint. This returns the **full diff** between PR head and base branch — no per-commit breakdown. When a merge commit is present, all files from the merged branch appear in the diff.

Since `owners-label` only adds labels and never removes them, the erroneous labels persist even after the user force-pushes to remove the merge commit.

## Code Paths

| What | Where | Role |
|---|---|---|
| Event handler | `owners-label.go:58-69` | Filters to opened/reopened/synchronize |
| File fetching | `owners-label.go:77-84` | Gets ALL changed files, maps to labels |
| Label application | `owners-label.go:109-116` | Only adds, never removes |
| DCO precedent | `dco.go:156-161` | `len(commit.Parents) > 1` to skip merges |
| GitHub client | `client.go:4616` | `ListPullRequestCommits` — returns commits with `Parents` |

## Recommended Solution

### Approach: Skip labeling when merge commits are present

Before processing files, call `ListPullRequestCommits()`. If any commit has `len(Parents) > 1`, return early. Labels are applied correctly on the subsequent `synchronize` event after force-push.

**Pros:**
- ~10-15 lines of code
- Follows DCO plugin pattern exactly
- One additional API call total
- Self-correcting: labels applied after force-push

**Cons:**
- Skips all labeling for that event (coarse-grained)
- Permanent merge commits = no labels ever (rare edge case)

### Why not danwinship's per-commit approach?

`ListPullRequestCommits` doesn't populate the `Files` field on `RepositoryCommit` (only `GetCommit` does, per `types.go:1354-1355`). The per-commit approach would require N additional `GetCommit` API calls — expensive and unnecessary for what is always a transient state that users fix by force-pushing.

### Implementation Checklist

- Add `ListPullRequestCommits` to the local `githubClient` interface in `owners-label.go`
- Add merge commit check early in `handle()`, before `GetPullRequestChanges`
- Log when skipping labeling due to merge commits
- Add test: PR with merge commits → no labels added
- Add test: PR without merge commits → labels added normally

## Labels to Apply

| Command | Rationale |
|---|---|
| `/area plugins` | Change is in `pkg/plugins/owners-label/` |
| `/remove-lifecycle rotten` | Issue is valid and actionable; reset triage-bot clock |
| `/good-first-issue` | Level 1: small scope, clear pattern, minimal expertise (was removed with stale lifecycle) |
| *(kind/feature already applied)* | — |

## Draft Comment

```
The root cause is that `owners-label` uses `GetPullRequestChanges` (the GitHub `GET /pulls/{n}/files` endpoint), which returns the full diff between the PR head and base branch without distinguishing which commits introduced which files. When a merge commit is present, all files from the merged branch appear in this diff, producing a flood of irrelevant labels. Since `owners-label` only adds labels and never removes them, these persist even after the merge commit is force-pushed away.

The simplest fix is to call `ListPullRequestCommits` before processing files and skip labeling entirely if any commit has multiple parents (i.e., is a merge commit). This pattern is already established in the DCO plugin (`pkg/plugins/dco/dco.go`), which uses `len(commit.Parents) > 1` to detect and skip merge commits. Labels would be applied correctly on the subsequent `synchronize` event after the user force-pushes to remove the merge commit. The per-commit file analysis approach discussed above would be more precise, but `ListPullRequestCommits` doesn't populate per-commit `Files` (only `GetCommit` does per the types), so it would require N additional API calls — unnecessarily expensive for what is always a transient state.

/area plugins
/good-first-issue
```

---
*Triage completed 2026-05-03 · Refreshed 2026-06-08*
