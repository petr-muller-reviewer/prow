---
pr: kubernetes-sigs/prow#1013
title: "feat: add squash-commits mode to cherrypicker"
head_sha: ce50c67292a53542103cf2b4318c3daf4c6a559f
base: main
reviewed_at: 2026-10-19T13:08:04Z
verdict: request-changes
gate:
  decision: hold
  gated_at: 2026-10-19T12:51:00Z
  gated_head_sha: ce50c67292a53542103cf2b4318c3daf4c6a559f
  reviewed_head_sha: ce50c67292a53542103cf2b4318c3daf4c6a559f
---

# Review

## Gate

Decision: **hold**. Three actionable findings remain in the current PR head. The public pkg/git/v2.Interactor interface also gained methods, which can break downstream implementations at compile time; confirm compatibility expectations or preserve the interface.

- Metadata lookup failures can still produce a squashed commit with missing or incorrect authorship (cmd/external-plugins/cherrypicker/server.go:813-827).
- Commit body text can suppress a real Co-authored-by trailer (cmd/external-plugins/cherrypicker/server.go:920-925).
- Long messages can exceed the OS argument limit when passed to Git (pkg/git/v2/interactor.go:439-449).
- The exported interface change affects downstream packages that implement Interactor or RepoClient (pkg/git/v2/interactor.go:60-64; pkg/git/v2/client_factory.go:84-88).

The earlier GitUser nil-pointer concern appears addressed by the new CommitWithAuthor path and its regression test. The --squash-commits flag defaults to false, so existing deployments retain their runtime behavior unless they enable it.

## Verdict

Request changes: the squash path can lose author attribution when GitHub metadata is unavailable, omit co-author trailers in some multi-commit messages, and fail for long commit messages. These are user-visible failures in the new mode.

## What this PR does

- Adds a default-off --squash-commits option to the cherrypicker.
- Fetches a pull request's net diff and applies it to the target branch with three-way support.
- Creates one signed-off commit for the cherry-pick and tries to preserve the original authorship.
- Leaves the existing per-commit git am path in place when squash mode is disabled.

## Findings

### [should-fix] Abort when commit metadata cannot be loaded
- where: cmd/external-plugins/cherrypicker/server.go:813-827
- concern: If listing the pull request's commits fails, the code logs a warning and continues with an empty commit list and author. It can then create a title-only commit attributed to the bot, losing the original authorship; stop before committing when required metadata is unavailable.
- excerpt: |
    commits, err := s.ghc.ListPullRequestCommits(org, repo, num)
    if err != nil {
        logger.WithError(err).Warnf("failed to list commits for #%d; squashed commit will omit authorship", num)
    }
    ...
    author := primaryAuthor(commits, prAuthorLogin)
    message := squashCommitMessage(title, commits, author)
    return r.CommitWithAuthor(message, author, true)

### [should-fix] Check existing co-author trailers in the trailer block
- where: cmd/external-plugins/cherrypicker/server.go:920-925
- concern: For a multi-commit PR, commit bodies are copied into the base message. If one body mentions a co-author trailer, this substring check treats it as already present and omits the real trailer, so GitHub does not credit that co-author.
- excerpt: |
    base = strings.TrimRight(base, "\n")
    lower := strings.ToLower(base)
    var add []string
    for _, t := range trailers {
        if !strings.Contains(lower, strings.ToLower(t)) {
            add = append(add, t)
        }
    }

### [should-fix] Pass long commit messages without an argv limit
- where: pkg/git/v2/interactor.go:439-449
- concern: The complete squashed message is passed as one --message argument. A sufficiently long pull request history can exceed Linux's per-argument limit and fail before Git runs; pass the message through a file or stdin.
- excerpt: |
    args := []string{"commit", "--message", message}
    if author != "" {
        args = append(args, "--author", author)
    }
    if signoff {
        args = append(args, "--signoff")
    }
    if out, err := i.executor.Run(args...); err != nil {
        return fmt.Errorf("error committing: %w. output: %s", err, string(out))
    }

## Checked

- --squash-commits defaults to false and the existing path still uses git am.
- The squash path applies the diff with git apply --3way --index and adds the bot's sign-off.
- The added end-to-end test checks PR creation but not the resulting commit author or message. Tests were not run.

## Open questions

- Can the squash operation stop before committing if author metadata cannot be loaded?
- Can trailer deduplication inspect only actual Git trailers rather than message-body text?
- Can the commit message be passed through a file or stdin so long pull requests still work?
