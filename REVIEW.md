---
pr: kubernetes-sigs/prow#1013
title: "feat: add squash-commits mode to cherrypicker"
head_sha: 399d1fd49b041dbf9b6d62d231a807b66bf99f1d
base: main
reviewed_at: 2026-10-20T12:27:33Z
verdict: needs-discussion
gate:
  decision: hold
  gated_at: 2026-10-20T12:43:39Z
  gated_head_sha: 399d1fd49b041dbf9b6d62d231a807b66bf99f1d
  reviewed_head_sha: 399d1fd49b041dbf9b6d62d231a807b66bf99f1d
refresh_log:
  - at: 2026-10-20T12:27:33Z
    old_sha: ce50c67292a53542103cf2b4318c3daf4c6a559f
    new_sha: 399d1fd49b041dbf9b6d62d231a807b66bf99f1d
    summary: "Resolved the three implementation findings with targeted fixes and a trailer regression test."
---

# Review

## Gate

Decision: **hold** at `399d1fd49b041dbf9b6d62d231a807b66bf99f1d`.

The previous implementation findings are addressed: metadata lookup failures stop the squash before applying the diff, co-author deduplication checks the final trailer block, and commit messages are passed through a file. The earlier nil-GitUser panic is also fixed. No outstanding implementation finding from the previous review remains.

One independent compatibility risk remains. This change adds `Apply` and `CommitWithAuthor` to the exported `pkg/git/v2.Interactor` interface (`interactor.go:63-67`), and the exported `RepoClient` embeds `Interactor` (`client_factory.go:92-95`). Downstream packages with their own implementations of either interface can therefore stop compiling. The squash flag defaults to false, so existing runtime behavior is unchanged by default, but that does not avoid the source compatibility break. Hold until maintainers confirm this interface break is acceptable or adjust the API to preserve downstream implementations.

### Findings status

- **Addressed:** metadata lookup failures, co-author trailer deduplication, long commit-message argument size, and nil author handling.
- **Open maintainer disposition:** source compatibility for external `Interactor` and `RepoClient` implementations.

### Merge risk

- The exported interface change may require downstream implementers to add two methods or prevent them from compiling against this release.
- The new squash mode is opt-in; the default remains the existing `git am` path.

## Verdict

Needs discussion: the three implementation findings from the previous review are resolved, but the source compatibility impact of extending the exported Interactor interface still needs maintainer disposition.

## What this PR does

- Adds a default-off --squash-commits option to the cherrypicker.
- Fetches a pull request's net diff and applies it to the target branch with three-way support.
- Creates one signed-off commit for the cherry-pick and tries to preserve the original authorship.
- Leaves the existing per-commit git am path in place when squash mode is disabled.

Since previous review:

- cmd/external-plugins/cherrypicker/server.go now aborts before applying the diff when either authorship metadata request fails, and reports those failures separately from conflicts.
- cmd/external-plugins/cherrypicker/server.go now deduplicates co-author trailers against only the final trailer block.
- pkg/git/v2/interactor.go writes the commit message to a temporary file and passes it with --file.

## Findings

No new findings in this refresh.

## Resolved

### [should-fix] Abort when commit metadata cannot be loaded
- where: cmd/external-plugins/cherrypicker/server.go:813-827
- concern: If listing the pull request's commits fails, the code logs a warning and continues with an empty commit list and author. It can then create a title-only commit attributed to the bot, losing the original authorship.
- excerpt: |
    commits, err := s.ghc.ListPullRequestCommits(org, repo, num)
    if err != nil {
        logger.WithError(err).Warnf("failed to list commits for #%d; squashed commit will omit authorship", num)
    }
    ...
    author := primaryAuthor(commits, prAuthorLogin)
    message := squashCommitMessage(title, commits, author)
    return r.CommitWithAuthor(message, author, true)
- resolution: The current code aborts when either ListPullRequestCommits or GetPullRequest fails, before applying the diff, and returns a metadata-specific response (server.go:697, 882-894).

### [should-fix] Check existing co-author trailers in the trailer block
- where: cmd/external-plugins/cherrypicker/server.go:920-925
- concern: For a multi-commit PR, commit bodies are copied into the base message. If one body mentions a co-author trailer, the substring check treats it as present and omits the actual trailer, so GitHub does not credit that co-author.
- excerpt: |
    lower := strings.ToLower(base)
    ...
    if !strings.Contains(lower, strings.ToLower(t)) {
        add = append(add, t)
    }
- resolution: Deduplication now scans only the final trailer block, and a regression test covers a co-author trailer inside a bullet body (server.go:1009-1050; server_test.go:1841-1855).

### [should-fix] Pass long commit messages without an argv limit
- where: pkg/git/v2/interactor.go:439-449
- concern: The complete squashed message was passed as one --message argument. A sufficiently long pull request history could exceed Linux's per-argument limit and fail before Git ran.
- excerpt: |
    args := []string{"commit", "--message", message}
    ...
    if out, err := i.executor.Run(args...); err != nil {
        return fmt.Errorf("error committing: %w. output: %s", err, string(out))
    }
- resolution: CommitWithAuthor now writes the message to a temporary file and invokes git commit with --file (interactor.go:472-489).

## Checked

- --squash-commits defaults to false and the existing path still uses git am.
- The squash path applies the diff with git apply --3way --index and adds the bot's sign-off.
- The author added a regression test for co-author trailers in commit bullet bodies. The metadata error and long-message paths have no new targeted test in this update. Tests were not run.

## Open questions

- Can downstream users that implement pkg/git/v2.Interactor remain source-compatible, or should the new operations use an optional capability interface?
