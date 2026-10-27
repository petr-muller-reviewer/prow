---
pr: kubernetes-sigs/prow#698
title: "feat: manage non-k8s members assigning issues"
head_sha: 5a0e394dfd2e0f56718965ddb31d576ec76612b5
base: main
reviewed_at: 2026-10-26T15:48:20Z
verdict: request-changes
refresh_log:
  - sha: 773c479cad28d019abfabee63bdba190d2544280
    refreshed_at: 2026-05-26T12:53:08Z
    summary: "No code changes; incorporated 5 new comments discussing design direction (TineoC, Amulyam24, lmktfy)"
  - sha: 773c479cad28d019abfabee63bdba190d2544280
    refreshed_at: 2026-05-30T23:35:02Z
    summary: "No code changes; 1 new comment from soltysh raising the member-assigns-non-member scenario"
  - sha: 773c479cad28d019abfabee63bdba190d2544280
    refreshed_at: 2026-07-26T23:00:46Z
    summary: "No code changes; 1 new comment from Amulyam24 clarifying GitHub's own assignee restrictions in response to soltysh's question"
  - sha: 5a0e394dfd2e0f56718965ddb31d576ec76612b5
    refreshed_at: 2026-10-26T15:48:20Z
    summary: "Full re-review after force-push 773c479 -> 5a0e394 rewrote the feature (+199/-6 vs main)"
---

# Review

## Verdict

**Request changes.** The rewrite resolves every blocker from the earlier review and the package tests pass, but the self-assignment policy can be bypassed by writing one's own login with different capitalization: the commenter lookup and the self-strip compare raw logins, while GitHub logins are case-insensitive. That should be normalized before merge; the remaining items are a maintainability nit and scope questions.

## What this PR does

- Blocks a non-member from self-assigning an issue that already has assignees, posting an explanation and a link to unassigned `triage/accepted` issues.
- Posts guidance (assignment still proceeds) when a non-member self-assigns an issue lacking `triage/accepted`, `help wanted`, or `good first issue`.
- Exempts PRs, org members, assignments of other users, and users already on the issue; membership and label lookups fail open.
- Replaces hardcoded `github.com` links and `CONTRIBUTING.md` probing with `github.DefaultHost` and `pkg/labels` constants, and isolates the policy in `checkSelfAssign()`.

Since the previous review (`773c479`): the author force-pushed `5a0e394`, replacing the `good-first-issue` gate with the acceptance-label policy, adding the block-on-assigned behavior, and adding tests for the new matrix.

## Findings

### [should-fix] Case-variant self login bypasses the policy
- where: `pkg/plugins/assign/assign.go:151`
- concern: `users` is keyed by the raw logins parsed from the comment (`parseLogins` preserves case), while the gate looks up `e.User.Login` case-sensitively. GitHub logins are case-insensitive, so a non-member writing `/assign @NewBie` (canonical login `newbie`) never enters `checkSelfAssign` and is assigned to an already-assigned issue without a comment, skipping the unaccepted-issue guidance too. The same comparison at line 198 also fails to strip the commenter from `toAdd`, so `/assign @NewBie @newbie` posts "I didn't assign you" and then assigns them anyway. Normalize with `github.NormLogin` when building and looking up `users`, and in the strip filter.
- excerpt: |
    if h.userType == "assignee(s)" && users[e.User.Login] {
    ...
        for _, login := range toAdd {
            if login != e.User.Login {

### [nit] `userType` display string doubles as a control-flow discriminator
- where: `pkg/plugins/assign/assign.go:151`
- concern: `handle()` is shared by the assign and reviewer handlers and decides whether to run this assign-only policy by comparing `h.userType` with the human-facing literal `"assignee(s)"`, which is also used in log messages. Rewording that label silently disables the policy with no compile error. A dedicated field (e.g. `appliesSelfAssignPolicy bool`) or moving the hook into the assign handler would make the intent explicit.
- excerpt: |
    if h.userType == "assignee(s)" && users[e.User.Login] {

## Resolved

Prior-review blockers addressed by the rewrite:

- Unused `GetRepo`/`GetFile` interface methods — removed; `GetFile`-based `CONTRIBUTING.md` probing dropped.
- Membership checked the wrong subject — now gated on the commenter being in `toAdd`, with an `isAssignee` early return; member-assigns-non-member is tested.
- `GetIssueLabels` error swallowed — membership and label lookups now fail open explicitly.
- Hardcoded `github.com` links — `github.DefaultHost` is used throughout.
- PR description/code mismatch — the PR body now describes blocking on assigned issues and fail-open behavior, matching the code.
- Policy ran on PRs — `e.IsPR` gate plus test.
- Inline logic in `handle()` — extracted into `checkSelfAssign()`.
- Membership checked after the label fetch — reordered.
- `"good-first-issue"` literal — replaced by `pkg/labels` constants.
- Test assertions keyed off `tc.name` — replaced by `labels`/`assignees`/`isPR` fields.

## Checked

- `e.Assignees` is populated for `issue_comment` events (`pkg/github/helpers.go:150`), so the assigned-issue check works on the real path.
- `go test ./pkg/plugins/assign/...` passes at `5a0e394`.
- Org members, non-self assignments, PRs, and already-assigned users bypass the policy as intended; a comment failure on the guidance path does not block assignment.
- The generated search link uses `url.QueryEscape` and `github.DefaultHost`.

## Open questions

- The algorithm quoted in the PR discussion had three rules; the third (flag repeated assignment attempts as contentious regardless of membership) is not implemented. Was it dropped from the agreed scope?
- Should the acceptance labels (`triage/accepted`, `help wanted`, `good first issue`) be configurable? They are Kubernetes conventions, and every Prow installation that enables the assign plugin gets this behavior and these links.
