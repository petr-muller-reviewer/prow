---
pr: kubernetes-sigs/prow#1013
title: "feat: add squash-commits mode to cherrypicker"
head_sha: 90697ba698cf1d87b59e524183ebfc321c5eecc0
base: main
reviewed_at: 2026-10-28T00:51:53Z
verdict: request-changes
checkout_sha: 340ff7c5d792b15f5a2757ce1574704d8fbd5491
gate:
  decision: do-not-merge
  gated_at: 2026-10-28T00:52:10Z
  gated_head_sha: 90697ba698cf1d87b59e524183ebfc321c5eecc0
  reviewed_head_sha: 90697ba698cf1d87b59e524183ebfc321c5eecc0
---

# Review

## Gate

Decision: **do-not-merge** at `90697ba698cf1d87b59e524183ebfc321c5eecc0` (2026-10-28T00:52:10Z).

The PR head is unchanged since the saved review, and the blocking temporary diff-file race remains. Requests for the same PR onto different target branches can execute concurrently but still write and remove the same filename, causing otherwise valid backports to fail. Prior GitHub review concerns about metadata failures, co-author trailers, message argument limits, binary contents, intermediate-commit conflicts, merge-commit PRs, and reverting base-only changes are addressed; the remaining suggestions do not independently block merging.

### Findings disposition

- **Not addressed; blocks merge — REVIEW.md, `cmd/external-plugins/cherrypicker/server.go:913-917`:** the filename contains only org, repo, and PR number, while request locks also include the target branch. Use a unique temporary file per invocation, close it before applying, and cover simultaneous requests for different targets.
- **Not addressed; non-blocking — REVIEW.md, `cmd/external-plugins/cherrypicker/server.go:701-711`:** preparation and commit failures still enter the apply-failure response and can create a manual-conflict issue. Improve classification and replace the unconditional `git am` error text.
- **Not addressed; non-blocking — REVIEW.md, `cmd/external-plugins/cherrypicker/server_test.go:1946-1949`, `1979-1982`, `2024-2037`:** one real PR commit is paired with two synthetic metadata entries, and final author/trailer/sign-off assertions remain absent.
- **Borderline; accepted follow-up — @thc1006, `cmd/external-plugins/cherrypicker/server.go:921`:** target-side renames remain unsupported by the net-diff apply path. The reviewer explicitly accepted a follow-up in the 2026-10-20T17:46:38Z review, and the PR description documents the limitation; this does not block merge.
- **Can't tell; compatibility decision — REVIEW.md, `pkg/git/v2/interactor.go:63-84`:** four new interface methods require updates in external implementations of `Interactor` or `RepoClient`. No affected downstream implementation or explicit maintainer acceptance was established; confirm the intended source-compatibility policy rather than treating this as a demonstrated deployment outage.

### Independent merge risk

- **API:** adding `Apply`, `CommitWithAuthor`, `MergeBase`, and `BinaryDiff` to exported `Interactor` also extends `RepoClient`. The blast radius is downstream code implementing either interface when rebuilt; callers using the supplied implementation need no change. The source-compatibility change is not protected by the runtime flag and is not described in the feature release note.
- **Configuration:** the only new flag is default-off `--squash-commits`; existing deployments retain the `git am` path. No schema migration, removed flag, or default change was found. Remove the new flag before rolling back to a binary that predates it.
- **Behavior:** enabling the process-wide flag changes all cherrypicks handled by that instance to one commit, rejects incomplete metadata including histories beyond the 250-commit API cap, and adds API/Git work per target. These risks affect opt-in instances; the feature is release-noted, while operator documentation should cover the cap, rename limitation, and manual fallback.
- The risk pass used direct code inspection and the existing review evidence. No listed repository skill specifically covers Go API/config compatibility, so no additional review skill was invoked.

### Gating list

1. **REVIEW.md blocking finding — `cmd/external-plugins/cherrypicker/server.go:913-917`:** isolate each invocation's diff file and verify concurrent requests cannot overwrite or delete one another's input. The current code is unchanged; this is the item driving **do-not-merge**.

## Verdict

Request changes: fix the shared temporary diff-file race before merging.

Code Quality and Deployment Risk independently identified the race, and verification confirmed that requests for different target branches can overwrite or delete each other's diff. The design otherwise looks reasonable, defaults remain unchanged, and both affected package test suites pass. Diagnostics and integration-test improvements are non-blocking; downstream interface compatibility needs an explicit maintainer decision, with no affected implementation established.

## What this PR does

- Adds a default-off `--squash-commits` flag to cherrypicker.
- Computes the PR's net change from the merge-base of its recorded base SHA and fetched head, including binary file contents.
- Applies that change with `git apply --3way --index` and creates one signed-off commit.
- Preserves primary authorship and collects co-author trailers, aborting when commit metadata is incomplete.
- Keeps the existing per-commit `git am` path when squash mode is disabled.

## Findings

### [blocking] Use a unique temporary diff file per request

- where: `cmd/external-plugins/cherrypicker/server.go:913-917`
- concern: CONFIRMED; Code Quality and Deployment Risk independently identified this race. Webhook handlers run concurrently and locks include the target branch, so requests for the same PR onto different branches can truncate or delete each other's diff and fail otherwise valid backports. Use `os.CreateTemp`, close the completed file before applying it, remove only that invocation's file, and add a regression test for concurrent squash requests to different targets.
- excerpt: |
    diffPath := fmt.Sprintf("/tmp/%s_%s_%d.squash.diff", org, repo, num)
    if err := os.WriteFile(diffPath, diff, 0644); err != nil {
        return fmt.Errorf("failed to write diff for PR #%d: %w", num, err)
    }
    defer os.Remove(diffPath)

### [should-fix] Distinguish preparation, apply, and commit failures

- where: `cmd/external-plugins/cherrypicker/server.go:701-704`
- concern: Non-blocking suggestion shared by Code Quality and Maintainability. New fetch, diff-generation, and commit failures enter this same apply-failure branch, which reports a failed `git am` and can create a manual-conflict issue when `issueOnConflict` is enabled. Use accurate operation descriptions and distinguish preparation and commit errors from an actual apply conflict.
- excerpt: |
    if err := applyErr; err != nil {
        errs := []error{fmt.Errorf("failed to `git am`: %w", err)}
        logger.WithError(err).Warn("failed to apply PR on top of target branch")
        resp := fmt.Sprintf("#%d failed to apply on top of branch %q:\n```\n%v\n```", num, targetBranch, err)

### [should-fix] Exercise real multi-commit history and final attribution

- where: `cmd/external-plugins/cherrypicker/server_test.go:1979-1982`
- concern: Non-blocking test improvement shared by Code Quality and Maintainability. The integration fixture creates only one actual PR commit but supplies two synthetic metadata entries, and its final assertions inspect commit count and file contents rather than author, co-author trailers, or sign-off. Create a genuine multi-commit history that rewrites the same lines, align metadata with it, and inspect the resulting commit's attribution.
- excerpt: |
    commits: []github.RepositoryCommit{
        commit("alice", "Alice", "alice@x.com", "bump magic number"),
        commit("bob", "Bob", "bob@x.com", "add comment"),
    },

### [question] Confirm the exported interface compatibility decision

- where: `pkg/git/v2/interactor.go:63-67`
- concern: Is adding four methods to exported `Interactor`, also embedded by `RepoClient`, acceptable for downstream implementations, or should the new operations use an optional capability interface? The added methods are confirmed, but no affected downstream implementation was identified; this is a conditional source-compatibility question rather than a demonstrated deployment break or required change.
- excerpt: |
    // Apply calls `git apply --3way --index`
    Apply(path string) error
    // CommitWithAuthor commits the already-staged index with the given message,
    // overriding the author, and optionally adding a Signed-off-by trailer.
    CommitWithAuthor(message, author string, signoff bool) error

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

- The local code matches PR head `90697ba698cf1d87b59e524183ebfc321c5eecc0`, excluding `REVIEW.md` and `REVIEW.html`; checkout HEAD `340ff7c5d792b15f5a2757ce1574704d8fbd5491` adds only review artifacts.
- All five changed files were reviewed from Code Quality, Maintainability, and Deployment Risk perspectives. The required change was checked against webhook goroutines, target-specific locks, and temporary-file creation/removal.
- `go test ./cmd/external-plugins/cherrypicker ./pkg/git/v2` passed. The Deployment Risk reviewer also reported 20 passing repetitions of the squash integration tests; these do not exercise the concurrent-request race.
- The default-off flag preserves the existing `git am` path. No configuration migration or new Kubernetes RBAC/GitHub permission requirement was identified.
- Metadata retrieval and truncation validation precede applying changes. Squash mode refuses incomplete commit metadata, including histories exceeding the 250-commit API cap.
- Integration coverage exercises an advancing base, a merge-commit PR, and a client without a GitUser getter. Commit messages use a file to avoid argument-size limits, and trailer deduplication checks the final trailer block.
- Git operations remain in `pkg/git/v2`, while GitHub-specific authorship and message policy remain in cherrypicker. Maintenance burden is MEDIUM; no independent design blocker was identified.
- Deployment risk is MEDIUM and concentrated in the opt-in path. Each target adds commit-list pagination, another PR metadata lookup, and two Git fetches; resource impact was not benchmarked.
- Operator documentation should explain the process-wide flag, metadata cap, manual fallback, and target-side rename limitation. Disable the flag to restore the existing runtime path; remove it before rolling back to an older binary.

## Open questions

- Is extending the exported `pkg/git/v2.Interactor` interface acceptable for downstream implementers, or should the new operations use an optional capability interface?
