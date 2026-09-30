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
