---
pr: kubernetes-sigs/prow#1077
title: "Process cherry-pick commands in PR reviews"
head_sha: 32418fa5ac8feb3d592ca4f8084cf10cf3bff98c
base: main
reviewed_at: 2026-11-19T15:28:03Z
verdict: approve
gate:
  decision: hold
  gated_at: 2026-11-19T15:28:03Z
  gated_head_sha: e065fe36e4ed8f54ca8881a622c05d7e34a83f48
  reviewed_head_sha: 32418fa5ac8feb3d592ca4f8084cf10cf3bff98c
---

# Review

## Gate

**hold** — two unaddressed should-fix items plus one policy question; no backward-incompatible changes.

Re-gated at `e065fe36e` (one follow-up commit, 2 files, +2/-9). That commit addresses both of @ivnovakov's inline comments: `handleReview` now checks only `ReviewActionSubmitted`, `TestReviewRequests` drops the synthetic pending/dismissed cases, and `TestCherryPickPRV2` now creates the `pending` branch so removing the replay filter fails with 10 PRs. Nothing else on the PR side is outstanding.

Gating list:

1. `cmd/external-plugins/cherrypicker/server.go:483-486` (REVIEW.md, should-fix) — a `ListReviews` failure still aborts the whole merge replay at `e065fe36e`, so a rate-limit blip drops comment- and label-based requests. Make review listing best-effort.
2. `site/content/en/docs/components/external-plugins/cherrypicker.md:34-43` (REVIEW.md, should-fix) — the example still points at `cherrypick` / `http://cherrypick`; existing deployments use `cherrypicker` / `http://cherrypicker`.
3. `cmd/external-plugins/cherrypicker/server.go:198-205`, `:487-491` (REVIEW.md, question) — dismissed reviews still supply commands at merge; confirm the policy or exclude `DISMISSED`.

Merge risk: no exported Go API (the plugin is `package main`), no CRD/proto, no config-schema or flag changes; existing `plugins.yaml` parses and older binaries debug-skip the new event type, so rollback is safe. Behavioral: merge-time review replay activates on the image bump alone for every repo already subscribed to `pull_request`; the release note says reviews are recognized but not that replay activates without a config change. Cost: one extra paginated `ListReviews` call per merged PR.

## Verdict

Solid, well-tested implementation; I would merge once the two should-fix items are handled and the dismissal policy is confirmed.

The live path reuses `handleIssueComment` end to end, so authorization, acknowledgement, base-branch rejection, closed-unmerged rejection and immediate processing on merged PRs all match conversation comments. Merge replay folds review bodies into the existing comment pipeline and sorts the combined stream by timestamp, which is the right shape. The remaining items are operational robustness (review-listing failures), a wrong operator example, and one deliberate policy choice that deserves an explicit yes.

## What this PR does

- Handles `pull_request_review` `submitted` events by converting the review body into a synthetic issue comment and delegating to the existing handler; `pending` reviews and `edited`/`dismissed` actions are ignored live.
- At merge, appends the bodies of submitted reviews (including dismissed ones) to the issue-comment list, sorts all sources by timestamp, and reuses the requester→branch dedup so the latest request supplies the response permalink and assignee.
- Moves chained-branch collection after the membership filter, so only an authorized requester's chain can influence a cherry-pick.
- Documents review bodies as a command source, the review-state policy, and the optional `pull_request_review` subscription needed for live handling.
- Extends the cherry-picker tests with live review states, merged-PR review handling, comment/review chronology, and review-listing failure coverage.

## Findings

### [should-fix] `ListReviews` failure drops the entire merge replay
- where: `cmd/external-plugins/cherrypicker/server.go:483-486`
- concern: a 403/secondary rate limit or transient 5xx now aborts replay for that merged PR, so comment- and label-initiated cherry-picks that worked before are lost with no retry (hook has already acked the delivery). Make review listing best-effort: log a warning and continue with the comments and labels already fetched; update `TestReviewListingError` accordingly.
- excerpt: |
    reviews, err := s.ghc.ListReviews(org, repo, num)
    if err != nil {
        return log, fmt.Errorf("failed to list reviews: %w", err)
    }

### [should-fix] Docs example targets a service that does not exist
- where: `site/content/en/docs/components/external-plugins/cherrypicker.md:34-43`
- concern: the example uses `name: cherrypick` / `endpoint: http://cherrypick`, while the subscriptions this note is aimed at (kueue, kjob, jobset, kernel-module-management) use `name: cherrypicker` / `endpoint: http://cherrypicker`. Copied into an existing entry, it registers a second plugin pointing at a service that does not exist. Use the real name/endpoint (or drop `endpoint`, which defaults to `http://{{name}}`) and note that `events: []` already receives all event types.
- excerpt: |
    external_plugins:
      org/repo:
      - name: cherrypick
        endpoint: http://cherrypick
        events:
        - issue_comment
        - pull_request
        - pull_request_review

### [nit] Replay parses every body twice, and the authorization rule lives only in the second pass
- where: `cmd/external-plugins/cherrypicker/server.go:506-558`
- concern: the first pass builds `requesterToComments` before membership filtering; the second pass re-parses every body and uses requester presence as the authorization check. Any future eligibility rule ("ignore bot users", dedup by comment ID) must be duplicated in both loops or the maps silently diverge. Reusing the parsed commands or collapsing to one post-authorization pass would make the invariant structural.
- excerpt: |
    // first look for our special comments
    for _, comment := range comments {
        commands := parseComment(comment)
    ...
    for _, comment := range comments {
        if _, allowed := requesterToComments[comment.User.Login]; !allowed {
            continue
        }
        for target, chain := range parseComment(comment) {

### [nit] The review-permalink assertion passes for the wrong reason
- where: `cmd/external-plugins/cherrypicker/server_test.go:817` (setup at `:661`)
- concern: the added `cherrypick/release-1.14` label can take over that branch's requester, so the response link for `release-1.14` is nondeterministic; the joined-comment check still passes because `release-1.16`'s response contains the review URL. Assert the response per branch instead of searching the concatenated blob.
- excerpt: |
    if !strings.Contains(strings.Join(ghc.comments, "\n"), "https://example.com/review/1") {
        t.Error("merge replay response missing review permalink")
    }

### [nit] Synthetic comment event has an undocumented coupling contract
- where: `cmd/external-plugins/cherrypicker/server.go:214-220`
- concern: `handleReview` fabricates an `IssueCommentEvent` with `PullRequest: &struct{}{}`, `Action: created`, and a comment with no ID, silently depending on `handleIssueComment`'s exact early validation and on `FormatICResponse` using only `Body`/`HTMLURL`/`User`. A short comment naming the synthetic fields would prevent drift (e.g. future comment-ID-based dedup would collapse every review to ID 0).
- excerpt: |
    return s.handleIssueComment(log, github.IssueCommentEvent{
        Action:  github.IssueCommentActionCreated,
        Repo:    event.Repo,
        Issue:   github.Issue{Number: event.PullRequest.Number, State: event.PullRequest.State, PullRequest: &struct{}{}},
        Comment: reviewComment(event.Review),
    })

### [nit] Pre-existing panic in `parseComment`, now reachable from reviews
- where: `cmd/external-plugins/cherrypicker/server.go:355-363`
- concern: a body ending in `/cherrypick   ` matches the regex with `match[1] == " "`, so `strings.Fields` returns an empty slice and `targetBranches[0]` panics; the handler runs in a goroutine with no recover, so it takes the process down. This is pre-existing for comments, but the PR adds review bodies as another way in; a `if len(targetBranches) == 0 { continue }` guard is a cheap follow-up.
- excerpt: |
    for _, match := range cherryPickRe.FindAllStringSubmatch(comment.Body, -1) {
        targetBranches := strings.Fields(match[1])
        targetBranch := targetBranches[0]

### [question] Should dismissed reviews still supply commands at merge?
- where: `cmd/external-plugins/cherrypicker/server.go:198-205`, `cmd/external-plugins/cherrypicker/server.go:487-491`
- concern: `submittedReview` counts `DISMISSED` for replay, but dismissal freezes the review body, so a `/cherrypick` in a dismissed review cannot be edited away and fires at merge. Prow's generic-comment mapping treats dismissal as deletion (`pkg/github/types.go:1363`), and a deleted issue comment is naturally absent from replay. The linked issue proposes this policy but explicitly asks for maintainer input; confirm it, or exclude `DISMISSED`.
- excerpt: |
    case github.ReviewStateApproved, github.ReviewStateChangesRequested, github.ReviewStateCommented, github.ReviewStateDismissed:
        return true

### [question] Live handling can race merge replay
- where: `cmd/external-plugins/cherrypicker/server.go:211-221`
- concern: if a submitted review is processed after the PR has merged (approval immediately followed by an auto-merge), `handleIssueComment` sees a merged PR and creates the cherry-pick, then replay re-attempts the same review/branch and posts a redundant "already cherry picked" or "pull request could not be created" response. The per-target lock prevents a duplicate PR, and issue comments share the race, but approvals are the typical merge trigger. Consider letting replay own pre-merge requests (would need `merged_at`, not currently parsed) or accept the duplicate response.
- excerpt: |
    func (s *Server) handleReview(log logrus.FieldLogger, event github.ReviewEvent) (logrus.FieldLogger, error) {
        if event.Action != github.ReviewActionSubmitted || !submittedReview(event.Review) || strings.EqualFold(string(event.Review.State), string(github.ReviewStateDismissed)) {
            return log, nil
        }

## Resolved

### [nit] One helper encoded two eligibility policies
- where: `cmd/external-plugins/cherrypicker/server.go:198-213`
- concern: `submittedReview` accepted `DISMISSED` while its only live caller immediately excluded it again, so the live-vs-replay split lived in two places.
- disposition: addressed in `e065fe36e` — `handleReview` now checks only `event.Action == github.ReviewActionSubmitted`, and the synthetic pending/dismissed state loop was removed from `TestReviewRequests`.

## Checked

- Live review commands go through `handleIssueComment`, so membership checks, open-PR acknowledgement, base-branch rejection, closed-unmerged rejection and immediate processing on merged PRs are the same code path as comments (`TestReviewRequests`, `TestReviewOnMergedPR`).
- `ListReviews` paginates in the prow client (`pkg/github/client.go:2747`), so merge replay does not silently lose older reviews on busy PRs.
- Hook already demuxes `pull_request_review` to external plugins listed under `events` (`pkg/hook/server.go:125`, `pkg/hook/server.go:185-221`), so the documented deployment step takes effect; entries with `events: []` already receive it.
- Chronological replay across comments and reviews works: the sort plus per-requester target map makes the newest request supply the permalink, and a later empty review or empty chain does not cancel an earlier command (`TestMergeReviewCommentChronology`, `testCherryPickPR`).
- Unauthorized targets and chains are dropped: the `fan` review/comment in `testCherryPickPR` contributes neither `unauthorized`, `unwanted` nor a chain.
- Pending reviews are excluded from replay; dismissed reviews are excluded live.
- No config schema, flag, CRD/proto or exported Go API changes; older binaries debug-skip `pull_request_review`, so rollback is safe.
- `gofmt` clean, `go vet` clean, `go test -p 1 ./cmd/external-plugins/cherrypicker/...` passes at `32418fa5a`.

## Open questions

- Chained branches are still keyed globally by target branch (`server.go:549-558`), not by requester, so a later authorized user's chain can attach to someone else's request for the same branch. Pre-existing and arguably what the chain means, but the PR is reworking this block.
- `ListReviews` adds one paginated API call to every merged PR, even in repos that never put commands in reviews. Probably fine through ghproxy, but worth a deliberate yes.
