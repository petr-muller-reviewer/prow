---
pr: kubernetes-sigs/prow#943
title: "cherrypicker: attribute cherrypick labels to the user who added them"
head_sha: b94a0f19abf206f247b9b4bd5817295e47eb2960
base: main
reviewed_at: 2026-09-22T00:31:46Z
verdict: request-changes
---

## Verdict

Request changes. The implementation can attribute labels applied directly in GitHub by a human, but it cannot identify the human who issued Prow's normal `/label cherrypick/...` command: Prow performs the GitHub label mutation as its bot. Reconstructing provenance at merge time also adds a paginated GitHub issue-events lookup and silently falls back to the PR author when it fails.

## What this PR does

- Adds label-event lookup to the cherrypicker GitHub client.
- Uses a label webhook sender, or an issue-event actor after merge, as the preferred requester.
- Rejects bot actors and otherwise falls back to the PR author.
- Adds unit coverage for direct human/bot actors and merged-PR label histories.

## Findings

### [blocking] Does not preserve the requester for Prow `/label` commands
- where: `cmd/external-plugins/cherrypicker/server.go:505-526; cmd/external-plugins/cherrypicker/server.go:570-575; pkg/plugins/label/label.go:256`
- concern: The label plugin authorizes the human comment author but calls `AddLabel` with Prow's GitHub credential. GitHub therefore reports the Prow bot as the label actor. `labelRequester` rejects that bot and falls back to the PR author, so the common `/label cherrypick/...` workflow neither assigns the cherry-pick to the requester nor preserves the intended authorization semantics for non-member PR authors. Carry the human requester at command-processing time, or retain the prior author-attribution behavior.
- excerpt: |
    // Labels are attributed to whoever added them, falling back to the PR author.
    events, err := s.ghc.ListIssueEvents(org, repo, num)
    ...
    if labelerOK && !s.isBot(labeler) {
        return labeler.Login, true
    }
    return prAuthor, labelerOK || isMember(prAuthor)

### [should-fix] Avoid merge-time issue-history scans for requester provenance
- where: `cmd/external-plugins/cherrypicker/server.go:505-516`
- concern: Every merged PR retaining a cherry-pick label now needs an issue-events API lookup, potentially scanning a paginated event history. This adds rate-limit and availability exposure; on failure the warning is followed by silent attribution to the PR author. Preserve the requester when the command is accepted instead of recovering it later from GitHub history.
- excerpt: |
    events, err := s.ghc.ListIssueEvents(org, repo, num)
    if err != nil {
        log.WithError(err).Warn("Failed to list issue events, attributing cherrypick labels to the PR author.")
    }

## Checked

- Direct GitHub human-label attribution, membership fallback, bot filtering, and re-label ordering are covered by the added table-driven tests.
- No configuration, manifest, flag, or migration behavior changes.
- The added client method matches the existing GitHub-client issue-events API shape.
- Targeted Go tests were started during review but did not complete before dependency compilation exceeded the execution window; no passing-test result is claimed.

## Open questions

- Is direct GitHub label application, rather than Prow's `/label` command, the only supported label-initiated cherry-pick workflow?
- If `/label` is supported, where should cherrypicker receive or durably retain the authenticated comment author without querying the full issue-event history at merge time?

