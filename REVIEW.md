---
pr: kubernetes-sigs/prow#782
title: "owners-label: add ignore_merge_commits config option"
head_sha: ddde196fedea6f58c0fd75fa102433444b989d21
base: main
reviewed_at: 2026-08-11T11:19:22Z
verdict: approve
refresh_log:
  - from_sha: f864b330e8bae580e7a1e177ed6c1b69259a279a
    to_sha: f864b330e8bae580e7a1e177ed6c1b69259a279a
    summary: No code changes. Prucek left an inline comment on config.go:258 reinforcing the existing config-granularity question.
  - from_sha: f864b330e8bae580e7a1e177ed6c1b69259a279a
    to_sha: ddde196fedea6f58c0fd75fa102433444b989d21
    summary: IgnoreMergeCommits converted from a global bool to a []string of org/org-repo entries with an IgnoreMergeCommitsFor(org, repo) helper, mirroring the existing SkipCollaborators pattern. Resolves the config-granularity question raised by all three review perspectives and by Prucek's inline comment.
  - from_sha: ddde196fedea6f58c0fd75fa102433444b989d21
    to_sha: ddde196fedea6f58c0fd75fa102433444b989d21
    summary: No code changes. Prucek flagged (1) generated docs are stale — plugin-config-documented.yaml doesn't reflect the new field, needs `make verify-codegen`; (2) the org/full membership-check loop is now duplicated a third time (MDYAMLRepos-style, SkipCollaborators, IgnoreMergeCommitsFor) and asked for it to be factored out.
---

## Summary

Adds opt-in `ignore_merge_commits` config to `Owners`. When enabled for a repo, `owners-label` calls `ListPullRequestCommits` and bails out if any commit has >1 parent. Prevents spurious label pollution when users accidentally push merge commits. Off by default, zero impact on existing deployments.

Since previous review:
- `IgnoreMergeCommits` changed from a global `bool` to a `[]string` of `org` / `org/repo` entries (`pkg/plugins/config.go:256-259`), following the same shape as `SkipCollaborators`.
- Added `Configuration.IgnoreMergeCommitsFor(org, repo)` (`pkg/plugins/config.go:305-313`), byte-for-byte the same lookup pattern as `SkipCollaborators`.
- `handlePullRequest` now calls `pc.PluginConfig.IgnoreMergeCommitsFor(pre.Repo.Owner.Login, pre.Repo.Name)` instead of reading the bool directly (`pkg/plugins/owners-label/owners-label.go:69`).
- No code changes since 2026-07-26T22:39:28Z. Prucek left two new comments on 2026-08-10 (see Findings): generated docs are stale, and the org/repo membership-check loop should be factored out now that it's duplicated three times.

## Findings

### [should-fix] Generated plugin-config-documented.yaml not regenerated
- where: `prow/cmd/generic-autobumper` docs / `pkg/plugins/plugin-config-documented.yaml` (generated)
- concern: Flagged by Prucek on 2026-08-10: the new `ignore_merge_commits` config field is not reflected in the generated documentation. `make verify-codegen` needs to be run and its output committed before merge, or CI's codegen-verify check will fail.

### [nit] Org/repo membership-check loop now duplicated a third time
- where: `pkg/plugins/config.go:284-289` (`MDYAMLRepos`-style helper), `:294-301` (`SkipCollaborators`), `:305-313` (`IgnoreMergeCommitsFor`)
- concern: Prucek requested (2026-08-10, inline on config.go:316) that the repeated `for _, elem := range list { if elem == org || elem == full { return true } }` pattern be factored into a shared helper now that it appears three times. The previous review's "Checked" section noted the pattern was reused but judged extraction unnecessary at two occurrences; a third occurrence changes that calculus and is worth a small helper, e.g. `func orgOrRepoInList(org, repo string, list []string) bool`.

### [should-fix] Merge commit API call runs before label-need check
- where: `pkg/plugins/owners-label/owners-label.go:77-88`
- concern: `ListPullRequestCommits` executes before `GetPullRequestChanges`. When no OWNERS labels apply, the function early-exits at line 99 with comment "Return now to save API tokens". The commit-listing call is wasted in that case. Moving the label-need check before the merge commit check avoids the extra API call.
- excerpt: |
    if ignoreMergeCommits {
        commits, err := ghc.ListPullRequestCommits(org, repo, number)
        ...
    }
    // later:
    if neededLabels.Len() == 0 {
        // No labels requested for the given files. Return now to save API tokens.
        return nil
    }

### [nit] No test coverage for IgnoreMergeCommitsFor / handlePullRequest wiring
- where: `pkg/plugins/config.go:305-313`, `pkg/plugins/owners-label/owners-label.go:69`
- concern: `IgnoreMergeCommitsFor` (new) and `handlePullRequest` (pre-existing, now calls it) have no direct test coverage — only the lower-level `handle()` is tested via `TestHandleIgnoreMergeCommits`, which still takes the resolved `bool`. Note `SkipCollaborators`, the pattern this mirrors, is likewise untested directly, so this matches existing convention rather than introducing a new gap.

### [nit] Add comment noting relationship with mergecommitblocker
- where: `pkg/plugins/owners-label/owners-label.go:77`
- concern: The feature exists specifically because of how `owners-label` and `mergecommitblocker` interact. A brief comment would save future maintainers from reconstructing this from the PR description. Also worth noting why this uses GitHub API rather than git (because `owners-label` does not clone the repo, unlike `mergecommitblocker`).

### [nit] Add error-path test for ListPullRequestCommits failure
- where: `pkg/plugins/owners-label/owners-label_test.go`
- concern: Happy paths are well-covered but no test verifies that a `ListPullRequestCommits` error is propagated rather than swallowed.

## Resolved

### [nit] Add comment on config field noting global scope — superseded in ddde196fe
- where: `pkg/plugins/config.go:256-259`
- original concern: A note like "This is a global setting; per-repo configuration is not currently supported" would set expectations for future contributors without having to trace the code.
- resolution: Moot — the field is no longer global-only (see below). The doc comment on `IgnoreMergeCommits` was updated to describe the org/org-repo list format instead.

### [question] Config granularity: global bool vs per-org/repo — resolved in ddde196fe
- where: `pkg/plugins/config.go:256-259`
- original concern: `MDYAMLRepos`, `SkipCollaborators`, and `Filenames` in the same `Owners` struct support per-org or per-repo configuration. `IgnoreMergeCommits` was a single bool for the entire prow instance. Multi-tenant deployments could not enable selectively. Flagged by three independent reviewers; reviewer Prucek also raised this as an inline PR comment on 2026-07-02.
- resolution: `IgnoreMergeCommits` is now `[]string` of `org` / `org/repo` entries, with `IgnoreMergeCommitsFor(org, repo)` resolving it the same way `SkipCollaborators` does. `handlePullRequest` was updated to call the new helper.

## Checked

- Merge commit detection logic (`len(commit.Parents) > 1`) is correct
- Error from `ListPullRequestCommits` is properly propagated with `%w` wrapping
- `githubClient` interface: real client and fake client both already implement `ListPullRequestCommits`
- `RepositoryCommit.Parents` exists as `[]GitCommit` in `pkg/github/types.go`
- Existing `TestHandle` passes `false` for new param, preserving prior behavior
- `TestHandleIgnoreMergeCommits` covers both paths (merge commit present / absent)
- No callers of `handle()` besides `handlePullRequest` and tests
- `handlePullRequest` gates on PR action before reaching new code
- No invariants lost; old behavior preserved when `ignoreMergeCommits=false`
- Checked reuse with `mergecommitblocker` (git-based, different approach) and `dco` (same expression but different purpose: filter vs gate). Neither warrants extraction.
- Checked reuse of the org/full membership-check loop itself across `Owners` config helpers: now duplicated three times (see Findings — Prucek requested extraction on 2026-08-10).
- Config field uses `omitempty`, defaults to `false`, existing configs parse identically
- No new permissions required; `ListPullRequestCommits` uses same GitHub token scope
- Upgrade and rollback both safe, no ordering dependencies

## Open questions

- Would reordering checks so label-need runs before merge-commit detection be acceptable? The existing code explicitly optimizes for the no-labels path.
