---
pr: kubernetes-sigs/prow#782
title: "owners-label: add ignore_merge_commits config option"
head_sha: 6a29c95931c3a76f016723bca5ee84eac2377bb6
base: main
reviewed_at: 2026-10-05T21:46:19Z
verdict: approve
gate:
  decision: hold
  gated_at: 2026-10-10T16:47:11Z
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
  - from_sha: 6a29c95931c3a76f016723bca5ee84eac2377bb6
    to_sha: 6a29c95931c3a76f016723bca5ee84eac2377bb6
    summary: No code changes. The author told Prucek on 2026-09-30 that the org/repo config scope and shared list lookup were addressed, and requested another look.
  - from_sha: 6a29c95931c3a76f016723bca5ee84eac2377bb6
    to_sha: 6a29c95931c3a76f016723bca5ee84eac2377bb6
    summary: No code changes. Prucek approved on 2026-10-01; the approval notifier reported that a pkg/plugins approver is still required.
---

## Gate

**Decision: hold.** Rechecked on 2026-10-10: the head remains `6a29c9593`, with no code or discussion changes since the previous gate. Prucek approved on 2026-10-01 after the config scope, generated docs, and duplicated lookup concerns were fixed. The local `should-fix` on API call ordering remains: configured repositories fetch every PR commit before checking whether any OWNERS labels apply. The call is paginated, so it can use multiple requests and can fail on a path that otherwise returns without further API work. Move it below the no-label return, or accept that cost and failure behavior explicitly, before merging.

### Gating list

- **Not addressed — `REVIEW.md`, `pkg/plugins/owners-label/owners-label.go:77-101`:** `ListPullRequestCommits` still runs before the `neededLabels.Len() == 0` early return. This is the sole substantive gate; move the call after that return or explicitly accept the added paginated requests and possible API error for PRs needing no labels.

### Independent merge risk

- `ignore_merge_commits` adds an optional `Owners` configuration field and exported Go struct field; there are no removals, signature changes, flag changes, or CRD changes. The new helper preserves the existing `MDYAMLEnabled` and `SkipCollaborators` matching rules. Deployments that do not list an org/repo retain prior behavior.
- Listed repositories skip label additions while a PR contains a merge commit, as intended. The new paginated commit lookup adds requests and a possible error before the no-label return in those repositories; its blast radius is limited to deployments that enable the option.

## Summary

Adds opt-in `ignore_merge_commits` config to `Owners`. When enabled for a repo, `owners-label` calls `ListPullRequestCommits` and bails out if any commit has >1 parent. Prevents spurious label pollution when users accidentally push merge commits. Off by default, zero impact on existing deployments.

Since previous review:
- `IgnoreMergeCommits` changed from a global `bool` to a `[]string` of `org` / `org/repo` entries (`pkg/plugins/config.go:256-259`), following the same shape as `SkipCollaborators`.
- Added `Configuration.IgnoreMergeCommitsFor(org, repo)` (`pkg/plugins/config.go:305-313`), byte-for-byte the same lookup pattern as `SkipCollaborators`.
- `handlePullRequest` now calls `pc.PluginConfig.IgnoreMergeCommitsFor(pre.Repo.Owner.Login, pre.Repo.Name)` instead of reading the bool directly (`pkg/plugins/owners-label/owners-label.go:69`).
- The branch was force-pushed/rebased from `ddde196fe` to `6a29c9593` on 2026-09-18. Its functional patch now changes five plugin paths (+185/-11): regenerated config documentation, a shared `orgRepoListed` helper, and table-driven coverage for all three org/repo-list accessors.
- This resolves Prucek's two 2026-08-10 comments: `plugin-config-documented.yaml` now includes `ignore_merge_commits`, and the previously triplicated membership check is shared.
- No code changes since the 2026-09-22 review. On 2026-09-30, carterpewpew told Prucek that the config scope and shared lookup concerns were addressed and requested another look; no new inline comments or reviews followed as of this refresh.
- No code changes since the 2026-09-30 refresh. Prucek approved on 2026-10-01 at 12:38 UTC. The approval notifier commented at 12:38 UTC that approval from a `pkg/plugins` approver is still needed; no new inline feedback was posted. The previously recorded gate concern about API call ordering remains unchanged.

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
