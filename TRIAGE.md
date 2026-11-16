---
issue: kubernetes-sigs/prow#666
title: "Allow use of Git subtrees with `mergecommitblocker`"
state: open
labels: kind/feature, area/plugins, lifecycle/rotten
main_sha: 3617afe983fa4df61a494fc2a60300367372de3a
triaged_at: 2026-11-16T22:39:21Z
verdict: accepted
refresh_log:
  - at: 2026-11-16T22:39:21Z
    since: 2026-10-28T01:17:51Z
    summary: "Bot moved the issue lifecycle/stale -> lifecycle/rotten; PR author RadaBDimitrova pinged @petr-muller for requested changes; main advanced 34 commits with no changes to mergecommitblocker, pkg/git/v2/interactor.go, or pkg/plugins/config.go; PR #719 still open and conflicting."
advice:
  advised_at: 2026-10-28T01:18:51Z
  based_on_triaged_at: 2026-10-28T01:17:51Z
---

# Triage

## Verdict

**accepted** — This is a valid feature request for Prow's `mergecommitblocker` plugin. The plugin currently blocks any PR containing a merge commit, including subtree merges; PR #719 proposes an opt-in, path-based exception and is open but needs a rebase.

## Resolution

Not resolved. PR #719 is open and has not been merged.

## What the issue reports

- Git subtree updates rely on merge commits to preserve history.
- `mergecommitblocker` currently flags every PR containing a merge commit.
- The issue asks for path-based exemptions so subtree merges can be used while retaining the blocker for other merge commits.

Since previous triage:
- No new technical information. `k8s-triage-robot` moved the issue from `lifecycle/stale` to `lifecycle/rotten` on 2026-11-16T07:01:58Z, which auto-closes after another 30 days of inactivity.
- RadaBDimitrova (PR #719 author) asked on 2026-11-16T09:47:25Z: "@petr-muller, are there any changes you would request for the PR?" — the same request-review blocker remains.
- Upstream `main` advanced 34 commits since `main_sha` (all dependency bumps); no diff in the `mergecommitblocker` plugin, `pkg/git/v2/interactor.go`, or `pkg/plugins/config.go`, so the cause finding still holds at current main.

## Findings

### [cause] The plugin blocks every merge commit in the PR range
- detail: The handler checks only whether any merge commit exists between the base and head; it does not inspect the files changed by each merge commit.
- evidence: `pkg/plugins/mergecommitblocker/mergecommitblocker.go:99-104`; `pkg/git/v2/interactor.go:590-599`.

### [related-code] The plugin has no path exemption configuration
- where: `pkg/plugins/mergecommitblocker/mergecommitblocker.go:47-53`
- excerpt: |
    // Only the Description field is specified because this plugin is not triggered with commands and is not configurable.
    return &pluginhelp.PluginHelp{
        Description: fmt.Sprintf("The merge commit blocker plugin adds the %s label to pull requests that contain merge commits", labels.MergeCommits),
    }, nil

### [related-code] Existing tests do not cover path-based exemptions
- where: `pkg/plugins/mergecommitblocker/mergecommitblocker_test.go:137-195`
- excerpt: |
    name: "merge commit label not exist, PR has merge commits"
    name: "merge commit label not exists, PR doesn't have merge commits"
- detail: Current tests cover merge-commit presence and label/comment behavior. PR #719 adds matching, non-matching, and mixed-path cases; its changed-file list does not include the documented plugin config file.

### [related-pr] PR #719 proposes the requested behavior
- ref: kubernetes-sigs/prow#719
- relevance: The open PR adds `excluded_paths` configuration and checks the changed paths of each merge commit. At triage time it has no formal reviews and the PR bot reports that it needs a rebase.
- status as of 2026-11-16: still OPEN, `mergeable_state: dirty` (`mergeable: false`), no reviews, head `941664570326bd59dfcbbdb62cc66f6c874f203f`, last updated 2026-10-20T07:05:24Z. Commits still date from 2026-05-13; no rebase since the previous triage.

### [related-issue] Issue #610 is related but not a duplicate
- ref: kubernetes-sigs/prow#610
- relevance: Closed feature request about `owners-label` retaining labels after merge commits; it concerns a different plugin behavior.

## Checked

- Read issue #666's description, labels, comments, and timeline cross-references.
- Searched all-state issues for `mergecommitblocker subtree`, `mergecommitblocker merge commits`, and `subtree merge commits`; no other matching issue appeared.
- Checked PRs referencing `666`; PR #719 is the related open implementation.
- Read the `mergecommitblocker` handler and tests, the v2 Git interactor method, nearby org/repo configuration lookup and regexp compilation patterns, and `pkg/plugins/plugin-config-documented.yaml`.
- Confirmed issue #610 is about `owners-label` and is not a duplicate.
- Refresh 2026-11-16: re-read issue #666 state, comments, and timeline since 2026-10-28T01:17:51Z; checked PR #719 state/reviews/files/commits; fetched `upstream` and diffed `3617afe..upstream/main` for the plugin, Git interactor, and plugin config files (no changes).

## Next steps

- Reply to RadaBDimitrova's 2026-11-16 ping on #666 with the concrete review requests, or rebase PR #719 first; the PR is still conflicting and unchanged since 2026-05-13.
- Review PR #719 after its rebase; verify per-merge path handling and config precedence.
- Add and validate the `merge_commit_blocker` configuration example in `pkg/plugins/plugin-config-documented.yaml`.
- Keep the issue open while the PR is under review; the issue is now `lifecycle/rotten`, so remove that label if the issue should not auto-close after another 30 days.

## Advice

1. **Ask the PR author to rebase PR #719.** The linked implementation remains open, has no formal review, and GitHub currently reports it as conflicting. This is the main blocker to reviewing the existing implementation.

```sh
gh pr comment 719 --repo kubernetes-sigs/prow --body "$(printf '%s\n' '> *This was generated by AI during triage.*' '' '@RadaBDimitrova, PR #719 is currently conflicting and needs a rebase before it can be reviewed. Please rebase it when you have a chance, or let us know if you need help.')"
```

2. **Mark the issue active and invite help if broader contribution would be useful.** The issue remains open with `lifecycle/stale`; the implementation PR is still conflicting. `kind/feature` and `area/plugins` are already applied, while `help wanted` is available but absent.

```sh
gh issue edit 666 --repo kubernetes-sigs/prow --remove-label lifecycle/stale --add-label lifecycle/active --add-label "help wanted"
```

## Open questions

- Would regex patterns, as proposed in PR #719, be the desired contract for excluded paths, or should the plugin use simpler glob patterns?
- Should organization-level exclusions be supported, with repository-level configuration taking precedence?
