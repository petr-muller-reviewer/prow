---
pr: kubernetes-sigs/prow#782
title: "owners-label: add ignore_merge_commits config option"
head_sha: 6a29c95931c3a76f016723bca5ee84eac2377bb6
base: main
reviewed_at: 2026-09-22T22:23:46Z
verdict: approve
gate:
  decision: hold
  gated_at: 2026-09-22T22:25:40Z
  gated_head_sha: 6a29c95931c3a76f016723bca5ee84eac2377bb6
  reviewed_head_sha: 6a29c95931c3a76f016723bca5ee84eac2377bb6
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
  - from_sha: ddde196fedea6f58c0fd75fa102433444b989d21
    to_sha: 6a29c95931c3a76f016723bca5ee84eac2377bb6
    summary: Rebased onto current main and incorporated prior feedback: regenerated plugin config docs, extracted the common org/repo-list lookup, and added direct coverage for all three lookup accessors.
---

## Gate

**Decision: hold.** The refreshed head is the reviewed head and prior reviewer feedback on configuration scope, generated documentation, and helper duplication is addressed. However, the remaining `should-fix` finding is unchanged: the opt-in merge-commit API call runs before the existing no-label early return, defeating that function's explicit API-token optimization for every configured PR without OWNERS labels.

### Gating list

- **[should-fix, REVIEW.md]** `pkg/plugins/owners-label/owners-label.go:77-101`: `ListPullRequestCommits` is invoked before `GetPullRequestChanges` computes `neededLabels`; move the merge-commit check after the `neededLabels.Len() == 0` return (or explicitly justify retaining the extra call) before merging.

### Independent merge risk

- `ignore_merge_commits` is an additive, opt-in `Owners` configuration field with `omitempty`; deployments not listing an org/repo retain current behavior. The only behavior change for listed repositories is intentionally suppressing label additions while a PR contains a merge commit. No exported API, flag, CRD, or default behavior changed.
- The current branch contains no commits after the refreshed review baseline (`6a29c9593`).

## Summary

Adds opt-in `ignore_merge_commits` config to `Owners`. When enabled for a repo, `owners-label` calls `ListPullRequestCommits` and bails out if any commit has >1 parent. Prevents spurious label pollution when users accidentally push merge commits. Off by default, zero impact on existing deployments.

Since previous review:
- `IgnoreMergeCommits` changed from a global `bool` to a `[]string` of `org` / `org/repo` entries (`pkg/plugins/config.go:256-259`), following the same shape as `SkipCollaborators`.
- Added `Configuration.IgnoreMergeCommitsFor(org, repo)` (`pkg/plugins/config.go:305-313`), byte-for-byte the same lookup pattern as `SkipCollaborators`.
- `handlePullRequest` now calls `pc.PluginConfig.IgnoreMergeCommitsFor(pre.Repo.Owner.Login, pre.Repo.Name)` instead of reading the bool directly (`pkg/plugins/owners-label/owners-label.go:69`).
- The branch was force-pushed/rebased from `ddde196fe` to `6a29c9593` on 2026-09-18. Its functional patch now changes five plugin paths (+185/-11): regenerated config documentation, a shared `orgRepoListed` helper, and table-driven coverage for all three org/repo-list accessors.
- This resolves Prucek's two 2026-08-10 comments: `plugin-config-documented.yaml` now includes `ignore_merge_commits`, and the previously triplicated membership check is shared.

## Findings

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

### [nit] Add comment noting relationship with mergecommitblocker
- where: `pkg/plugins/owners-label/owners-label.go:77`
- concern: The feature exists specifically because of how `owners-label` and `mergecommitblocker` interact. A brief comment would save future maintainers from reconstructing this from the PR description. Also worth noting why this uses GitHub API rather than git (because `owners-label` does not clone the repo, unlike `mergecommitblocker`).

### [nit] Add error-path test for ListPullRequestCommits failure
- where: `pkg/plugins/owners-label/owners-label_test.go`
- concern: Happy paths are well-covered but no test verifies that a `ListPullRequestCommits` error is propagated rather than swallowed.

## Resolved

### [should-fix] Generated plugin-config-documented.yaml not regenerated — resolved in 6a29c9593
- where: `pkg/plugins/plugin-config-documented.yaml:503-508`
- original concern: The generated plugin configuration documentation did not include `ignore_merge_commits`, and `make verify-codegen` would fail.
- resolution: The regenerated documentation now includes the field and its org/org-repo semantics.

### [nit] Org/repo membership-check loop duplicated three times — resolved in 6a29c9593
- where: `pkg/plugins/config.go:317-340`
- original concern: `MDYAMLEnabled`, `SkipCollaborators`, and `IgnoreMergeCommitsFor` each implemented the same org/full-repository membership loop.
- resolution: All three delegate to the new shared `orgRepoListed` helper.

### [nit] No direct coverage for IgnoreMergeCommitsFor — resolved in 6a29c9593
- where: `pkg/plugins/config_test.go:154-222`
- original concern: The config accessor was only indirectly exercised through the lower-level owners-label handler.
- resolution: `TestOwnersOrgRepoLists` exercises `MDYAMLEnabled`, `SkipCollaborators`, and `IgnoreMergeCommitsFor` against empty, org, org/repo, and non-matching lists.

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
- The shared `orgRepoListed` helper preserves the prior exact-match behavior for all three `Owners` config accessors.
- Config field uses `omitempty`, defaults to `false`, existing configs parse identically
- No new permissions required; `ListPullRequestCommits` uses same GitHub token scope
- Upgrade and rollback both safe, no ordering dependencies

## Open questions

- Would reordering checks so label-need runs before merge-commit detection be acceptable? The existing code explicitly optimizes for the no-labels path.
