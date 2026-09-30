---
pr: kubernetes-sigs/prow#638
title: "Automate adding kind/* labels to cherry-pick PRs"
head_sha: 9404b76f07618de9757274ca3797b46e2d51e76a
base: main
reviewed_at: 2026-07-27T00:26:12Z
verdict: approve
---

## Findings

### [should-fix] redundant GetIssueLabels call per cherry-pick target branch
- where: `cmd/external-plugins/cherrypicker/server.go:586-590`
- concern: `handle()` calls `s.ghc.GetIssueLabels(org, repo, num)` again even though the caller (`handlePullRequest`/`handlePullRequestLabelAdded`, ~line 390) already fetched the same labels for the same PR. Since `handle()` runs once per target branch, this re-fetches identical data from GitHub on every branch iteration for a chained cherry-pick. Thread the already-fetched labels/kindLabels into `handle()` as a parameter instead of re-querying.
- excerpt: |
    // server.go ~390 (existing call in caller)
    labels, err := s.ghc.GetIssueLabels(org, repo, num)
    ...
    // server.go ~586 (new, duplicate call inside handle())
    labels, err := s.ghc.GetIssueLabels(org, repo, num)
    if err != nil {
        logger.WithError(err).Debug("failed to get issue labels")
    }

### [should-fix] label-fetch failure logged at Debug, not Warn
- where: `cmd/external-plugins/cherrypicker/server.go:587`
- concern: the new `GetIssueLabels` error is logged at `Debug`, while a similar non-fatal "don't fail the operation but surface the problem" case elsewhere in this file (~line 622, "failed to assign to new PR") uses `Warn`. A real API failure here (e.g. a token-permission regression) would silently and invisibly drop `/kind` lines from every cherry-pick with no signal in default-level logs. Bump to `Warn`.

### [should-fix] no unit test for kindLabelsFromIssueLabels edge cases
- where: `cmd/external-plugins/cherrypicker/server.go:733-744`
- concern: the function's documented edge cases (a label exactly `"kind/"` with empty suffix, duplicate `kind/x` labels) are only exercised indirectly via one end-to-end `server_test.go` case with two well-formed labels. Add a small table-driven test directly against this pure function.

### [nit] no test for the GetIssueLabels error path in handle()
- where: `cmd/external-plugins/cherrypicker/server.go` (`handle`)
- concern: no test verifies the cherry-pick PR is still created successfully (without `/kind` lines) when the label fetch fails — the one new branch of behavior added to `handle()`.

### [nit] redundant sort in lib.go
- where: `cmd/external-plugins/cherrypicker/lib/lib.go:37-46`
- concern: `kindLabels` is sorted here even though the only production caller (`kindLabelsFromIssueLabels`, via `sets.List`) already returns a sorted slice. Defensible as defensive robustness for other future callers, not a bug.

### [question] CreateCherrypickBody's growing positional parameter list
- where: `cmd/external-plugins/cherrypicker/lib/lib.go`
- concern: this exported function's signature keeps growing positionally (release notes, chain branches, now kindLabels) as more "copy X from parent PR" features get added. Worth an options struct next time a field is added — not blocking now.

## Checked

- No config struct, YAML/JSON field, CLI flag, or ProwJob semantics changes — purely additive behavior, low deployment risk.
- Label-fetch failure degrades gracefully: cherry-pick still proceeds without `/kind` lines rather than aborting.
- Dedup/sort of kind labels via `sets.New`/`sets.List` is deterministic and idiomatic, consistent with existing usage in the file.
- `HelpProvider` text updated alongside the behavior change.
- Existing `server_test.go` case updated for the new parameter/happy path; unrelated `bugzilla_test.go` call site correctly updated for the new signature; bugzilla cherry-pick body regex detection confirmed unaffected by the new `/kind` lines.
- `CreateCherrypickBody`'s signature change is internal-only (all in-tree call sites updated), not a public API break.

## Open questions

- In repos without kind-label automation configured, or without a matching `kind/*` label, the emitted `/kind <name>` command could produce a bot error/reply comment on the new cherry-pick PR — was this considered, and is it worth a one-line release-note mention?
- Was the duplicate `GetIssueLabels` call (already fetched by the caller) an intentional simplification, or just missed when threading data into `handle()`?

## Followups

### Reuse parent PR labels across cherry-pick targets
- category: cleanup
- necessity: should — avoid repeated GitHub API calls for the same source PR, especially when one event creates multiple cherry-picks.
- where: `cmd/external-plugins/cherrypicker/server.go` (`handlePullRequest`, `handleIssueComment`, `handle`); `cmd/external-plugins/cherrypicker/server_test.go`
- why followup: PR #638 added a label fetch in `handle()` to keep kind-label copying local to PR creation. The merged-PR path already has the same labels, so an event can now fetch them again for every target branch; this efficiency fix did not block the merge.
- handoff prompt:
  ```text
  In kubernetes-sigs/prow, following merged PR #638, "Automate adding kind/* labels to cherry-pick PRs" (merge commit bcf4297e528a75b2aa580c5dce3e7b97e38f4553), remove redundant source-PR label fetches in cmd/external-plugins/cherrypicker/server.go. Work from the current merged default branch. handlePullRequest already calls GetIssueLabels before iterating target branches; reuse those labels (or normalized kind names) when building every cherry-pick PR. For a merged PR triggered by an issue comment, obtain labels at most once after the request has passed validation and reuse them across all targets. Preserve the existing handling of mandatory label-fetch failure in handlePullRequest and the best-effort behavior for an optional fetch in the comment path. Add a focused multi-target test in server_test.go that checks the expected PR bodies and GetIssueLabels call count for both event paths. Keep PR creation, branch selection, and non-kind labels otherwise unchanged.
  ```

### Test kind-label edge cases and surface fetch failures
- category: reliability
- necessity: should — protect the new label-copy behavior and make silent omissions visible.
- where: `cmd/external-plugins/cherrypicker/server.go` (`kindLabelsFromIssueLabels`, label-fetch error path); `cmd/external-plugins/cherrypicker/server_test.go`
- why followup: PR #638 covers only well-formed labels in one end-to-end case; its documented empty-suffix and duplicate handling and its best-effort label-fetch failure path are untested. The failure is logged only at Debug. These are narrow reliability improvements after merge.
- handoff prompt:
  ```text
  In kubernetes-sigs/prow, following merged PR #638, "Automate adding kind/* labels to cherry-pick PRs" (merge commit bcf4297e528a75b2aa580c5dce3e7b97e38f4553), strengthen kind-label handling in cmd/external-plugins/cherrypicker/server.go and server_test.go. Work from the current merged default branch; if the earlier followup to reuse source-PR labels has landed, test its remaining optional fetch path. Add table-driven tests of kindLabelsFromIssueLabels covering non-kind labels, `kind/` with an empty suffix, duplicate kind labels, and sorted output. Make the fake GitHub client able to return a GetIssueLabels error, then test that a comment-driven cherry-pick still opens a PR without `/kind` lines when that optional fetch fails. Log that omission at Warn with the underlying error so operators can diagnose it. Keep the operation best-effort and avoid broader cherrypicker error-handling changes.
  ```
