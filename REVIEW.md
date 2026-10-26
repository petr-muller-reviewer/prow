---
pr: 698
title: "feat: manage non-k8s members assigning issues"
head_sha: 773c479cad28d019abfabee63bdba190d2544280
base: main
reviewed_at: 2026-07-26T23:00:46Z
verdict: REQUEST_CHANGES
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
---

# PR #698: feat: manage non-k8s members assigning issues

**Author:** TineoC | **Files:** `assign.go` (+53/-2), `assign_test.go` (+77/-3)
**Reviewer verdicts:** Code Quality: REQUEST_CHANGES · Maintainability: REQUEST_CHANGES · Deployment Risk: MEDIUM

## Summary

Post an educational comment when a non-org-member uses `/assign` on an issue without the `good-first-issue` label, directing them to beginner-friendly issues. Labels → membership → CONTRIBUTING.md lookup → comment with links. Assignment still proceeds regardless.

### Since previous review (2026-05-26)

- **TineoC** (author) linked a concrete motivation: [kubernetes/kubernetes#131724](https://github.com/kubernetes/kubernetes/issues/131724#issuecomment-3619565989) where 10+ non-members self-assigned a single issue in quick succession by exploiting the "has commented" loophole that allows assignment.
- **Amulyam24** raised the design concern independently: the restriction may be too broad for `help-wanted` issues or minor bugs not labeled `good-first-issue`. Suggested a per-org/per-repo feature gate since `/assign` is commonly enabled at org level.
- **lmktfy** proposed a more nuanced algorithm: (1) if an issue is already assigned, non-members can't reassign it; (2) if a non-member tries to assign an unaccepted issue, send an educational message but clarify work isn't forbidden; (3) if multiple users have already attempted assignment, flag it as contentious regardless of membership. **Amulyam24 endorsed this approach.**

These comments strengthen the "no opt-out mechanism" concern and the "PR description vs. code mismatch" concern, and surface a new design question about whether `good-first-issue` is the right gate or whether issue acceptance state / existing assignment should be the signal.

- **soltysh** (2026-05-26, 14:15) — asked how this works when an org member assigns a non-org member to any issue. Notes from experience there are legitimate cases where a member needs to assign a non-member. This directly validates our **"Checks the commenter, not the target assignees"** blocker — the current code would incorrectly fire for this case since it only checks `e.User.Login`.

### Since previous review (2026-05-30)

- **Amulyam24** (2026-06-10) answered soltysh's question with GitHub's own current behavior: assigning a non-member/non-collaborator who hasn't commented already fails outright, with GitHub returning an explicit "GitHub didn't allow me to assign the following users" error and a link to the contributor guide. This means the "org member assigns a non-member" case soltysh raised is largely pre-filtered by GitHub's own assignee API restriction rather than reachable through this plugin's code path — it doesn't resolve the **"Checks the commenter, not the target assignees"** blocker (a non-member who *has* commented, and is thus assignable, can still be assigned by an org member without tripping the educational message incorrectly on the member), but it narrows the practical blast radius of that gap.

## Maintainer Advisor — Final Recommendation: REQUEST_CHANGES

All three reviewers independently identified overlapping correctness and design issues. Two logic bugs (error swallowing on label fetch and wrong membership check target) cause the educational message to fire incorrectly in real-world scenarios. Combined with an unused interface method flagged by all three reviewers, these are functional defects that need fixing before merge.

## Converging Concerns (flagged by 2+ reviewers)

### Unused `GetRepo` in interface [3/3 reviewers] — BLOCKER
Added to `githubClient` but never called in production code. Widens the interface contract and forces all implementations and mocks to satisfy it for no reason. Remove it.
> `assign.go:76`

### Wrong membership check target (self-assign guard missing) [2/3 reviewers] — BLOCKER
`IsMember(org, e.User.Login)` checks the commenter, not the assignees. `/assign @org-member` by a non-member fires the message incorrectly. `/assign @non-member` by an org member skips it. The check must be scoped to self-assignment only — verify the commenter is actually in `toAdd`.
> `assign.go:167`

### `GetIssueLabels` error swallowed — false positives [2/3 reviewers] — BLOCKER
When label fetch fails, `hasGoodFirstIssue` defaults to `false` and the code proceeds to check membership and potentially post an incorrect educational comment. Should fail open: skip the educational path entirely on error.
> `assign.go:154-156`

### Hardcoded `github.com` URLs [3/3 reviewers] — BLOCKER
Generated comment links use hardcoded `github.com` instead of `github.DefaultHost`. Prow installations on GitHub Enterprise get broken links. The default URL comparison at line 186 also creates hidden coupling to the help plugin's default.
> `assign.go:180, 186, 190`

## Additional Blockers

### PR description vs. code behavior mismatch
PR says "restrict self-assignment for non-organization members" but code still calls `h.add()` unconditionally — the user gets assigned and just sees a comment. Either the description is wrong or the code should skip assignment. Clarify intent.
> `assign.go:197`

### Triggers on PRs unnecessarily
`userType == "assignee(s)"` doesn't exclude PRs. `/assign` on a PR calls `GetIssueLabels` + `IsMember` + `GetFile` for no reason — PRs don't carry `good-first-issue` labels. Gate on `!e.IsPR`.
> `assign.go:153`

## Concerns

- **1–4 extra API calls per `/assign`, not cached:** `GetIssueLabels` on every assign, then conditionally `IsMember` + up to 2x `GetFile`. Consider checking membership first (often cached) to avoid label fetch for org members.
- **No opt-out mechanism:** Every Prow installation using the assign plugin gets this behavior upon upgrade. No per-org or per-repo configuration flag.
- **~45 lines of logic inlined in `handle()`** (`assign.go:153-196`): Breaks the generic handler's symmetry via `h.userType == "assignee(s)"` (display string as type discriminator). Should be extracted into a dedicated function or wired as a callback like `addFailureResponse`.
- **`userType` string comparison for control flow** (`assign.go:153`): Using a display-oriented string as a type discriminator is fragile. Use a dedicated boolean or enum field on the `handler` struct.

## Nits & Test Gaps

- **`isMemberCalled` assertions keyed off `tc.name` strings** [2/3 reviewers] (`assign_test.go:513-522`): Add `expectIsMemberCalled *bool` to the test case struct and check inside the loop.
- **Test setup uses `tc.name` for conditional state** [2/3 reviewers] (`assign_test.go:450`): Add a `labels` field to the test case struct so each case is self-contained.
- **Missing test coverage:** No test for (1) `CONTRIBUTING.md` not found + `HelpGuidelinesURL` fallback, (2) `/assign @other-user` by a non-member, (3) `GetIssueLabels` error, (4) non-nil `config`.
- **Hardcoded `"good-first-issue"` label:** No constant, no comment. Define as a package-level constant.
- **Message tone:** "It looks like you're new!" may feel patronizing to contributors from other orgs.

## Deployment Notes

- Each `/assign` now makes 1–4 additional GitHub API calls. Not cached. At scale, increases API token consumption.
- No opt-out mechanism — all orgs using the assign plugin get this behavior once deployed.
- Hardcoded `github.com` URLs produce broken links for GitHub Enterprise installations.
- Rollback is safe — reverting removes the educational messages with no state to clean up.

## Required Changes Checklist

- [ ] Remove unused `GetRepo` from interface
- [ ] Fix self-assignment guard — only fire when commenter is in `toAdd`
- [ ] Fail open on `GetIssueLabels` error — skip educational flow
- [ ] Use `github.DefaultHost` instead of hardcoded `github.com`
- [ ] Clarify: should assignment be blocked or just warned? Fix PR description accordingly
- [ ] Gate on `!e.IsPR` to avoid unnecessary API calls on pull requests
- [ ] Extract educational message logic into a dedicated function
- [ ] Add configuration flag to enable/disable behavior per-org/repo
- [ ] Reorder: check membership before labels to reduce API calls for members
- [ ] Define `"good-first-issue"` as a named constant
- [ ] Restructure test assertions to use struct fields instead of `tc.name`
- [ ] Add test cases for fallback paths and error conditions
