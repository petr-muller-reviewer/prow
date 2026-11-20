---
issue: kubernetes-sigs/prow#1076
title: "Support /cherrypick commands in pull request review bodies"
state: open
labels: 
main_sha: 54944683d1f737a9ff8c7615ccf8fd6655d1580f
triaged_at: 2026-11-20T00:45:22Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/feature, area/plugins]
---

# Triage

## Verdict

**Accepted** — a valid feature request for the cherrypicker external plugin, already implemented by open PR #1077; what remains is review and maintainer sign-off on the review-lifecycle policy.

The plugin recognizes `/cherrypick` only in issue comments and the PR body, using a single regex that already works on any text. Review bodies are dropped twice over: `pull_request_review` is not dispatched, and merge replay never lists reviews. All the GitHub plumbing (types, client method, webhook dispatch precedent) already exists in `pkg/github` and `pkg/hook`, so the change is localized and additive. Effort is Level 2 (moderate): one external plugin plus a per-repo subscription change in test-infra, touching merge-time behavior for every cherrypicker deployment, with policy questions (dismissed reviews, edited bodies) that need a maintainer decision.

## What the issue reports

- `/cherrypick` commands written in submitted pull request review bodies are never processed, whether the review is submitted on an open PR or the PR merges later.
- The event dispatcher handles `issue_comment` and `pull_request` but not `pull_request_review`; the merge handler collects issue comments, the PR body, and labels but not review bodies.
- Repositories also omit `pull_request_review` from their cherrypicker subscription, which blocks live handling independent of the handler.
- Proposed policy: handle a submitted review like a created conversation comment; ignore pending reviews; edits and dismissals do not trigger processing; merge-time collection uses current bodies of all submitted reviews including dismissed ones; inline review comments are out of scope.
- Motivated by kubernetes-sigs/kueue#17859, where review-body requests were silently ignored and had to be reposted as a comment.

## Findings

### [cause] pull_request_review events never reach the plugin

- detail: `handleEvent` switches on `issue_comment` and `pull_request` only; every other event type falls through to a debug log, so a submitted review is dropped even when the webhook is delivered.
- evidence: `cmd/external-plugins/cherrypicker/server.go:155-185`

```go
	switch eventType {
	case "issue_comment":
	...
	case "pull_request":
	...
	default:
		l.Debugf("skipping event of type %q", eventType)
	}
```

### [cause] merge replay ignores review bodies

- detail: `handlePullRequestClosed` lists issue comments, prepends the PR body as a synthetic comment, then builds `requester -> target branch -> comment` and chain maps. It never calls `ListReviews`, so review-body commands are lost at merge despite the handler running.
- evidence: `cmd/external-plugins/cherrypicker/server.go:423-472`

```go
	comments, err := s.ghc.ListIssueComments(org, repo, num)
	...
	// treat PR body/description as a comment
	comments = append([]github.IssueComment{{
		Body: body,
		User: pr.User,
	}}, comments...)
```

### [related-code] the comment path owns all request semantics

- where: `cmd/external-plugins/cherrypicker/server.go:187-317`
- excerpt: `handleIssueComment` implements command parsing, `IsMember` authorization, open-PR acknowledgement, closed-unmerged rejection, and immediate processing on merged PRs — everything a review request must reuse.

```go
	// PR is not yet done, inform the user we will remember their instructions and create PRs later.
	if ic.Issue.State != "closed" {
```

### [related-code] review API plumbing already exists

- where: `pkg/github/types.go:1037-1073`, `pkg/github/client.go:2747`, `pkg/hook/server.go:126`, `pkg/plugins/lgtm/lgtm.go:168`
- detail: `ReviewEvent`, `Review` with `SubmittedAt`, and the `APPROVED`/`CHANGES_REQUESTED`/`COMMENTED`/`DISMISSED`/`PENDING` states are defined; `ListReviews` is implemented on the client; `pkg/hook` already dispatches review events to plugins, with lgtm as precedent. Only cherrypicker's local `githubClient` interface and switch lack reviews.

### [related-code] the command regex needs no change

- where: `cmd/external-plugins/cherrypicker/server.go:45`
- excerpt: `var cherryPickRe = regexp.MustCompile(`(?m)^(?:/cherrypick|/cherry-pick)\s+(.+)$`)` — line-anchored and applied to arbitrary text, so review bodies parse unchanged.

### [related-code] test coverage is partial

- where: `cmd/external-plugins/cherrypicker/server_test.go:311` (`testCherryPickIC`), `:489` (`testCherryPickPR`)
- detail: live comment requests and merge replay with comments, PR body, labels, and authorization are covered. The fake `fghc` holds no review data, so pending, dismissed, empty-body, edited-body, and cross-source dedup cases are untested.

### [related-issue] motivating incident

- ref: kubernetes-sigs/kueue#17859
- relevance: a review contained `/cherrypick release-0.20` and `/cherrypick release-0.19` before merge; both were ignored. A later conversation comment repeating the commands triggered the bot, producing kueue#18839 for release-0.20, while release-0.19 hit a separate content conflict.

### [related-pr] implementation PR

- ref: kubernetes-sigs/prow#1077
- relevance: implements live review handling and merge replay (3 files, +253/−9, tests and docs). It is open, non-draft, mergeable, green on unit, race, integration, lint, and image-build jobs, and needs `lgtm`/`approve`.

## Resolved

None yet — the issue is open, unlabeled, and has no comments.

## Checked

- Duplicate search across all states for `cherrypick review body`, `cherrypicker pull_request_review`, `cherrypick review comment`, `cherrypicker review`: only this issue. #1011 (squashed diff) and #544 (abort on first failure) are unrelated.
- Issue timeline cross-references: only PR #1077; no linked issues, no comments, no labels.
- kueue's cherrypicker subscription in test-infra `config/prow/plugins.yaml:2119-2126` lists `issue_comment` and `pull_request` only, confirming the deployment gap.
- A `pull_request` `closed` event is sufficient for merge-time collection, so review replay needs no subscription change; only live acknowledgement does.
- Ruled out a parser change: `cherryPickRe` already matches review-body text, so `parseComment` is reusable as-is.
- Ruled out a GitHub API limitation: listing reviews returns current bodies of submitted and dismissed reviews, and pending drafts are distinguishable by state.
- Considered and rejected a review-specific handler that replies in the review thread: it duplicates authorization and open/merged/closed logic for no functional gain unless maintainers insist review feedback not appear as a PR comment.

## Next steps

- Review PR #1077; it is green and needs `lgtm`/`approve` to merge.
- Decide the review-lifecycle policy: whether commands in dismissed reviews still fire at merge (PR #1077's choice) or dismissal revokes the request, and whether the body at merge time is authoritative.
- Recommend labels `kind/feature` and `area/plugins`; the issue currently has none.
- After the handler ships, land a test-infra change adding `pull_request_review` to cherrypicker subscriptions (starting with `kubernetes-sigs/kueue`) for live acknowledgement.
- Decide whether #1077's chained-branch authorization tightening (only authorized requesters contribute chains) stays bundled or is split out.

## Open questions

- "Should a `/cherrypick` in a dismissed review still trigger at merge, or should dismissal retract the request?"
- "Is editing the review body before merge the intended way to retract a request, or should edits and dismissals cancel an already-acknowledged command?"
- "Should the chained-branch authorization tightening in #1077 land in the same PR, or be split out?"
