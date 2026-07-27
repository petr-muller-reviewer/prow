---
pr: 614
title: "cherrypicker: comment when label is ignored for non-org member"
head_sha: 182bd6e6ba9f08336b58c2f87688fd516657c74f
base: main
reviewed_at: 2026-04-14
verdict: MERGE
---

# Review: kubernetes-sigs/prow#614 — cherrypicker: comment when label is ignored for non-org member

- Author: valen-mascarenhas14
- Size: L (+187 / −25)
- Fixes: #468
- Files changed: `cmd/external-plugins/cherrypicker/server.go`, `cmd/external-plugins/cherrypicker/server_test.go`

## Merge gate decision

**MERGE — CLEAR TO MERGE**

Gate check passed — no show-stoppers found (2026-04-14, 2nd pass).

Second gate pass (2026-04-14) confirms the earlier finding: all three reviewers independently
concluded APPROVE with no critical issues. The PR adds a well-tested, low-risk behavioral
change and includes a clean refactoring that improves readability. No bugs, regressions, or
deployment-breaking changes were identified. No new concerns since the first gate check.

Non-blocking observations:

- The non-member's cherry-pick label is not removed after rejection, which could be mildly
  confusing. Pre-existing behavior, worth a follow-up issue.
- `IsMember` vs `ListOrgMembers` inconsistency between the two code paths is now more visible
  after the refactor; worth noting for future caching work.

## Overview

Refactors `handlePullRequest` into two focused handlers (`handlePullRequestLabelAdded`,
`handlePullRequestClosed`) and adds an explicit GitHub comment when a cherry-pick label is
ignored because the PR author is not an org member. Previously this was silent.

The refactor also removes the `foundCherryPickComments` / `foundCherryPickLabels` bookkeeping
in the closed-PR path, replacing it with a simpler `len(requesterToComments) == 0` check.

## Reviewer perspectives

| Perspective | Verdict | Note |
|---|---|---|
| Code Quality (senior Go engineer) | COMMENT | — |
| Maintainability | APPROVE | Burden: LOW |
| Deployment Risk | LOW RISK | No breaking changes |

## Converging concerns

Issues independently flagged by two or more reviewers carry the most weight.

### [converging] Non-member rejection swallows `createComment` error
Flagged by: Code Quality + Maintainability

In `handlePullRequestLabelAdded`, if `createComment` fails, the error is logged but the
function returns `nil`. In the comment-based path (`handleIssueComment`), equivalent errors
are returned. A transient GitHub API failure here means the rejection comment is silently
lost with no retry. Inconsistency makes debugging harder.

`server.go:366-370`

### [converging] `PullRequestActionOpened` silently dropped
Flagged by: Code Quality + Maintainability

The original code accepted this action but immediately bailed at the `!pr.Merged` check
(opened PRs are never merged). Dropping it is functionally a no-op, but the removal is a
behavioral change not mentioned in the PR description. A brief note in the description or a
code comment would help.

### [converging] Inconsistent membership checking: `IsMember` vs `ListOrgMembers`
Flagged by: Code Quality + Maintainability (noted positively by Deployment Risk)

The label path uses `IsMember(org, prAuthor)` (single-user API call) while the closed path
uses `ListOrgMembers()` (fetches all members). The `IsMember` approach is actually better for
API rate limits at scale, and is consistent with `handleIssueComment`. However, a code comment
should explain why the PR author is checked (GitHub does not expose who added a label in the
`PullRequestEvent` payload).

`server.go:360-372` vs `server.go:475`

## What looks good

- **Clean structural split** — Splitting by action into dedicated handlers is easier to follow
  than the original monolith with interleaved conditionals. Each handler has a single, clear
  responsibility. Removes confusing `foundCherryPickComments` / `foundCherryPickLabels` boolean
  flags.
- **Correct simplification in `handlePullRequestClosed`** — Moving the
  `requesterToComments[pr.User.Login]` map init inside the label loop and replacing the two
  boolean flags with `len(requesterToComments) == 0` is equivalent and simpler. A subtle
  correctness fix that avoids creating the inner map when there are no cherry-pick labels.
- **Handles unmerged PRs gracefully** — The new `handlePullRequestLabelAdded` responds to
  labels on unmerged PRs with a "will cherry-pick once merged" message. Previously, labels on
  unmerged PRs were silently dropped. Good UX improvement.
- **Reuses existing `notOrgMemberMessageTemplate`** — Consistent messaging with the
  comment-based cherry-pick path. No new templates or string formats introduced.
- **Well-scoped tests** — `TestHandlePullRequestLabelAdded_BaseEqualsTarget` and
  `TestHandlePullRequestLabelAdded_NonMemberRejected` test behavior, not implementation
  details. Existing `testCherryPickPRWithLabels` properly updated to set `event.Label` for
  labeled events — fixing a pre-existing test gap.
- **Better API efficiency for label events** — Using `IsMember` (one API call) instead of
  `ListOrgMembers` (potentially paginated) for the label path is a positive change for GitHub
  API rate limits at scale. Label events now also process only the triggering label rather
  than re-scanning all labels, reducing duplicate processing.

## Issues

### 1. [medium] Rejected label is not removed from the PR
When a non-member's label is rejected, the label stays on the PR. If the PR later merges,
`handlePullRequestClosed` picks up that label again and silently drops it (filtered by
`ListOrgMembers`). This works but is confusing for users — the label lingers despite the
rejection comment.

Consider removing the label after posting the rejection comment. Same issue applies when
`targetBranch == baseBranch`.

`server.go:366-371`

### 2. [nit] Missing trailing newline in `server_test.go`
The diff shows `\ No newline at end of file`. Should be fixed.

`server_test.go:1461`

### 3. [nit] Inconsistent `botUser` setup in new tests
`TestHandlePullRequestLabelAdded_BaseEqualsTarget` sets `botUser` on the server, but
`TestHandlePullRequestLabelAdded_NonMemberRejected` does not. Neither path uses it, so both
should omit it for consistency.

## Suggestions

- **Add code comment at `IsMember` call** — Explain that the label event payload does not
  identify who added the label, so checking the PR author is the best available proxy. This
  prevents future maintainers from "fixing" something that is an intentional trade-off.
  (`server.go:360-372`)
- **Propagate `createComment` error** — Return the error from `createComment` instead of
  swallowing it, for consistency with other error handling paths. If swallowing is
  intentional (e.g., to avoid retries that would re-trigger the label event), add a comment
  explaining why. (`server.go:366-370`)
- **Remove rejected label after posting comment**:
  ```go
   if !ok {
       resp := fmt.Sprintf(notOrgMemberMessageTemplate, org, org, org, prAuthor)
       if err := s.createComment(log, org, repo, num, nil, resp); err != nil {
           log.WithError(err).WithField("response", resp).Error("Failed to create comment.")
       }
  +    if err := s.ghc.RemoveLabel(org, repo, num, pre.Label.Name); err != nil {
  +        log.WithError(err).Error("Failed to remove rejected cherry-pick label.")
  +    }
       return log, nil
   }
  ```
- **Update PR description** — Mention the removal of `PullRequestActionOpened` handling and
  why it was safe to drop. This helps reviewers and future changelog readers.

## Advisor verdict

**APPROVE WITH SUGGESTIONS**

All three reviewers agree this is a well-structured, low-risk change that improves both code
organization and user experience. No critical or blocking issues were identified. The
concerns raised are real but non-blocking: edge-case UX imperfections and minor code hygiene
items, not correctness bugs or deployment risks.

The core idea is sound and the refactoring improves readability. The missing trailing newline
should be fixed before merge. The label removal, error handling, and code comment suggestions
are worth discussing with the author. The rest are minor.

## Review checklist

- [ ] Trailing newline restored in `server_test.go`
- [ ] Add code comment explaining `IsMember` on PR author (GitHub API limitation)
- [ ] Discuss: propagate `createComment` error or document why it is swallowed?
- [ ] Discuss: remove rejected label after posting comment?
- [ ] Update PR description to note `PullRequestActionOpened` removal
- [ ] Consistent `botUser` setup in new tests
- [ ] Tests pass (`go test ./cmd/external-plugins/cherrypicker/...`)

## Followups

PR #614 merged as `0ea29c45b5093eb6085cc1f6f777e927918cc65f` on 2026-04-20. The following
post-merge followups were identified and accepted on 2026-04-27.

### 1. [deferred-review, should] Unify membership-checking strategy between the label-added and merge-processing paths

**Where:** `cmd/external-plugins/cherrypicker/server.go` — `handlePullRequestLabelAdded` vs `handlePullRequestClosed`

`handlePullRequestLabelAdded` (new in #614) checks membership with a single `IsMember(org, user)`
call. `handlePullRequestClosed` (pre-existing) still checks membership by fetching the entire
org roster via `ListOrgMembers(org, "all")` and scanning it per requester. petr-muller flagged
this inconsistency twice during review ("Could be an opportunity for a followup - but pls not
in this PR, it's already a bit involved to review" / "fair enough. could be worth a followup")
and explicitly deferred it rather than block the PR.

```
In kubernetes-sigs/prow, following PR #614 ("external-plugins/cherrypicker: comment when label
is ignored for non-org member", merged as 0ea29c45b5093eb6085cc1f6f777e927918cc65f), unify the
membership-checking strategy in cmd/external-plugins/cherrypicker/server.go.

Context: handlePullRequestLabelAdded checks membership with a single ghc.IsMember(org, prAuthor)
call. handlePullRequestClosed still checks membership by calling ghc.ListOrgMembers(org, "all")
and scanning the full result with slices.ContainsFunc for each requester in requesterToComments.
This was called out during PR #614's review as an intentionally deferred followup (the reviewer
did not want an additional refactor in that PR).

Task: In handlePullRequestClosed, replace the ListOrgMembers(org, "all") + slices.ContainsFunc
scan with a per-requester ghc.IsMember(org, requester) call, matching the approach already used
in handlePullRequestLabelAdded. Preserve existing behavior: requesters that are not members must
still be deleted from requesterToComments before cherry-picks are processed. Remove the now-unused
ListOrgMembers call/import if nothing else in the file needs it.

Acceptance criteria:
- handlePullRequestClosed no longer calls ListOrgMembers; it calls IsMember per requester instead.
- Existing tests for handlePullRequestClosed (including non-member rejection scenarios) still pass.
- Add/update a test asserting a non-member's merge-time cherry-pick request is still filtered out
  under the new IsMember-based check.
- `go test ./cmd/external-plugins/cherrypicker/...` passes.

Out of scope: changing the label-added path, changing behavior for s.allowAll, any comment/label
UX changes (that's a separate followup for the same PR).
```

### 2. [tech-debt, should] Remove the rejected cherry-pick label after a non-org-member rejection

**Where:** `cmd/external-plugins/cherrypicker/server.go` — `handlePullRequestLabelAdded`, non-member branch (originally noted at `server.go:366-371`)

When a non-member's cherry-pick label triggers a rejection comment, the `cherrypick/XXX` label
itself is never removed from the PR — `handlePullRequestLabelAdded` only posts a comment and
returns. If the PR later merges, `handlePullRequestClosed` re-derives cherry-pick intent from
scratch (current comments + current labels), picks up the same still-attached label again, keyed
by the PR author, runs its own membership check, and silently deletes that requester entry — no
second comment, no visible action. Net effect: no bug, but the label lingers on the PR forever
with no further explanation, reading as "nothing happened" rather than "permanently declined",
and the merge-time pass burns an extra scan/API round for a foregone conclusion. This was raised
as a non-blocking observation during review.

```
In kubernetes-sigs/prow, following PR #614 ("external-plugins/cherrypicker: comment when label
is ignored for non-org member", merged as 0ea29c45b5093eb6085cc1f6f777e927918cc65f), fix a UX
rough edge in cmd/external-plugins/cherrypicker/server.go's handlePullRequestLabelAdded.

Context: when a cherry-pick label is added by/for a non-org-member PR author,
handlePullRequestLabelAdded posts a rejection comment (using notOrgMemberMessageTemplate) but
never removes the triggering label. The label stays on the PR. If the PR is later merged,
handlePullRequestClosed re-scans PR labels, finds the same cherrypick/XXX label still attached,
and silently filters it out again during its own membership check — with no further comment.
The label is effectively permanently ignored but never cleaned up or explained.

Task: in the non-member branch of handlePullRequestLabelAdded (immediately after posting the
rejection comment via s.createComment), remove the triggering label from the PR via
s.ghc.RemoveLabel(org, repo, num, pre.Label.Name). On error, log it (WithError) rather than
returning it — mirror the existing "log, don't fail" treatment of the createComment error just
above it, since the label removal is best-effort cleanup, not the primary action.

Acceptance criteria:
- After a non-member rejection, the label is removed via RemoveLabel in addition to the existing
  comment.
- A RemoveLabel failure is logged and does not change the function's return value (log, nil).
- Add a test (extending or alongside TestHandlePullRequestLabelAdded_NonMemberRejected) that
  asserts RemoveLabel was called with the correct org/repo/num/label after rejection.
- `go test ./cmd/external-plugins/cherrypicker/...` passes.

Out of scope: the targetBranch == baseBranch rejection path (same PR's suggestion mentioned this
case too, but scope this followup to the non-member rejection only unless trivially shared code
makes it natural to cover both); the membership-check unification (that's a separate followup).
```

### 3. [deferred-review, could] Wire PR-description cherry-pick commands into the `Opened` event, not just merge-time replay

**Where:** `cmd/external-plugins/cherrypicker/server.go` — historical `PullRequestActionOpened` handling (discussed at `server.go:338`)

During review, petr-muller asked why `PullRequestActionOpened` handling was dropped. Investigating
history, he found it was added long ago so PR authors could embed `/cherrypick release-X` commands
directly in the PR description at open time — but it never actually triggered anything immediately
(it always bailed on the `!pr.Merged` check), so removing it in PR #614 was a functional no-op. He
noted: "If we wanted to fix this we'd probably need to wire the comment handler behavior to the PR
opened event (no need to do it here in this PR)." PR-description commands still only take effect
at merge time via `handlePullRequestClosed`'s comment-replay, with no early feedback (e.g. no
immediate "you're not a member" comment) the way labels now get.

```
In kubernetes-sigs/prow, following PR #614 ("external-plugins/cherrypicker: comment when label
is ignored for non-org member", merged as 0ea29c45b5093eb6085cc1f6f777e927918cc65f), consider
extending early-feedback handling in cmd/external-plugins/cherrypicker/server.go to PR-description
cherry-pick commands, not just labels.

Context: PR authors can embed /cherrypick release-X commands directly in a PR description. This
has existed for a long time but only ever takes effect at merge time, via handlePullRequestClosed
replaying the PR body as a synthetic comment through parseComment. Unlike the label path added in
PR #614 (handlePullRequestLabelAdded), there is no immediate feedback when the PR is opened: no
early membership check, no early rejection comment if the author isn't an org member. This was
flagged during PR #614's review as a pre-existing, unrelated gap — explicitly out of scope for
that PR, and speculative rather than requested by anyone as urgent.

Task: add a handler for github.PullRequestActionOpened that parses the PR body for /cherrypick
commands via parseComment, and applies the same immediate-feedback treatment as
handlePullRequestLabelAdded: an early IsMember-based membership check (respecting s.allowAll)
with a notOrgMemberMessageTemplate-based rejection comment if the author is not a member, and a
"once this merges, I will cherry-pick" acknowledgement comment otherwise (mirroring the
handlePullRequestLabelAdded flow). Actual cherry-pick execution should remain deferred to merge
time via handlePullRequestClosed's existing comment-replay path, unchanged.

Acceptance criteria:
- Opening a PR whose description contains a valid /cherrypick command produces an immediate
  comment (rejection or acknowledgement) without waiting for merge.
- No duplicate cherry-pick or duplicate comment occurs at merge time for the same command.
- New tests cover: non-member author with a /cherrypick command in the description at open time;
  member author with a /cherrypick command in the description at open time.
- `go test ./cmd/external-plugins/cherrypicker/...` passes.

Out of scope: changing comment-based /cherrypick handling for already-open PRs (handleIssueComment
is unaffected); changing label-based handling; any change to merge-time cherry-pick execution
itself, only what happens at Opened time before merge.
```

---
Generated 2026-04-09, gate checks 2026-04-10 & 2026-04-14 — kubernetes-sigs/prow#614 — Multi-perspective maintainer review
