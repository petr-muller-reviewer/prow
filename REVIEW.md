---
pr: kubernetes-sigs/prow#943
title: "cherrypicker: attribute cherrypick labels to the user who added them"
head_sha: b94a0f19abf206f247b9b4bd5817295e47eb2960
base: main
reviewed_at: 2026-10-16T22:49:12Z
verdict: request-changes
refresh_log:
  - old_sha: b94a0f19abf206f247b9b4bd5817295e47eb2960
    new_sha: b94a0f19abf206f247b9b4bd5817295e47eb2960
    summary: "No code changes; recorded approval activity and merge."
---

## Verdict

Request changes. The implementation can attribute labels applied directly in GitHub by a human, but it cannot identify the human who issued Prow's normal `/label cherrypick/...` command: Prow performs the GitHub label mutation as its bot. Reconstructing provenance at merge time also adds a paginated GitHub issue-events lookup and silently falls back to the PR author when it fails.

## What this PR does

- Adds label-event lookup to the cherrypicker GitHub client.
- Uses a label webhook sender, or an issue-event actor after merge, as the preferred requester.
- Rejects bot actors and otherwise falls back to the PR author.
- Adds unit coverage for direct human/bot actors and merged-PR label histories.

Since previous review:

- No code changes: `b94a0f19abf206f247b9b4bd5817295e47eb2960` remains the PR head.
- `matthyx` approved the PR on 2026-10-16T15:26:28Z; the PR merged via `app/kubernetes-prow` on 2026-10-16T15:48:12Z.

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

## Followups

Accepted: 2. Skipped: 0. PR merged as `4425e386ffba859f8fbfb535617559f05d37bf2f`.

### 1. Fix `/label` requester attribution and authorization

- Category: correctness/security
- Necessity: must
- Mode: deliver
- Where: `cmd/external-plugins/cherrypicker/server.go:343-583`; `pkg/plugins/label/label.go:150-256,314-347`
- Why followup: The blocking attribution finding merged unfixed. A bot actor also supplies authorization when it is an org member, even though the human issuing an unrestricted `/label` command need not be a member. This authorization concern depends on deployment label configuration and bot membership.

```text
In kubernetes-sigs/prow, implement a post-merge followup to PR #943, "cherrypicker: attribute cherrypick labels to the user who added them", merged as 4425e386ffba859f8fbfb535617559f05d37bf2f. Work from the merged default branch and inspect current code before applying changes.

Mode: deliver. Fix requester attribution and authorization for Prow-mediated /label cherrypick/<branch> requests. Focus on cmd/external-plugins/cherrypicker/server.go and server_test.go, plus pkg/plugins/label/label.go and the relevant command/event boundary.

The label plugin applies labels with Prow's GitHub credential, so a label webhook or issue-event actor identifies the bot, not the human command author. labelRequester currently avoids assigning to the bot but can still authorize using the bot's org membership. With an unrestricted additional cherry-pick label, a non-member commenter can therefore cause a request to be accepted through a member bot. Verify this conditional path and preserve the authenticated human requester's identity and authorization decision through label application and later merge processing. Choose a durable, restart-safe handoff supported by the existing architecture; do not infer the human's identity from a bot label event or correlate by timing alone.

Acceptance criteria: an authorized human's /label request is attributed and assigned to that human for both pre-merge and already-merged PRs; an unauthorized non-member cannot gain authorization solely through a member bot when allowAll is false and the PR author is a non-member; existing intentionally supported member-author fallback and allowAll behavior remain explicit and tested; direct human label application still works. Add regression coverage that models a human command followed by a bot label event, including member/non-member authors and command users, and verify identity survives the merge handoff. Run the relevant cherrypicker and label-plugin tests and report the results.

Scope guard: keep changes limited to cherry-pick requester identity, its authorization, and the minimum handoff required. Do not introduce a general-purpose storage service, unrelated label-plugin policy changes, or broad refactoring. If the current architecture cannot supply a trustworthy durable identity, explain the concrete constraint rather than introducing speculative attribution. Coordinate the provenance design with followup 2, which reduces event-history lookups. Do not publish externally without an explicit request.
```

### 2. Reduce reliance on GitHub event history

- Category: reliability/performance
- Necessity: should
- Mode: deliver
- Depends on: followup 1's requester provenance design
- Where: `cmd/external-plugins/cherrypicker/server.go:505-526`; `pkg/github/client.go:4768-4801`
- Why followup: The issue-history finding merged unfixed. Each applicable merge fetches potentially paginated history; lookup failure may misattribute to a member author or drop a member-requested cherry-pick on a non-member author's PR.

```text
In kubernetes-sigs/prow, implement a post-merge followup to PR #943, "cherrypicker: attribute cherrypick labels to the user who added them", merged as 4425e386ffba859f8fbfb535617559f05d37bf2f. Work from the merged default branch and inspect the current implementation. This task depends on the durable human-requester handoff established by followup 1, "Fix /label requester attribution and authorization"; reuse that design rather than creating competing provenance mechanisms.

Mode: deliver. Reduce merge-time GitHub issue-event-history lookups in cmd/external-plugins/cherrypicker/server.go and add focused coverage in server_test.go. The merged implementation calls ListIssueEvents whenever cherry-pick labels are present, scans the complete paginated history, and falls back to the PR author on failure. That can either change the assignee or silently drop a member's request when the PR author is not an org member.

Use the trustworthy requester information recorded for Prow-mediated requests to avoid history scans when provenance is already available. Define explicit recovery behavior for unresolved provenance, including labels applied directly in GitHub and event-API failures. Preserve authorization and direct-label support; make any unavoidable failed recovery or skipped request diagnosable using the component's existing logging/comment conventions. Do not promise constant API cost for direct-label provenance if the available GitHub interface cannot provide it.

Acceptance criteria: merge handling for requests with valid recorded provenance does not call ListIssueEvents solely to rediscover the requester; direct human labeling remains supported; tests cover mixed recorded/direct-label requests, missing provenance, relabeling, and event-API failure for member and non-member PR authors; failures neither invent requester identities nor authorize using an untrusted actor, and any skipped request is diagnosable. Assert API-call behavior with the fake client and run the relevant cherrypicker tests. Document any remaining history dependency and its tradeoff.

Scope guard: reuse followup 1's handoff, keep existing GitHub-client pagination semantics for other callers, and avoid unrelated cache, metrics, storage, or client refactoring. If followup 1 has not landed, identify that dependency and coordinate the design before implementing a parallel solution. Do not publish externally without an explicit request.
```

