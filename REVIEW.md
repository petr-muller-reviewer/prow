---
pr: 607
title: "No longer check at mentions in commit messages and PR title"
head_sha: 9e8160a9ebc24ecee5df99c62d74068e765cf6b9
base: main
reviewed_at: "2026-06-22"
verdict: approve-with-suggestions
---

# PR #607: No longer check at mentions in commit messages and PR title

**Author:** [@XiShanYongYe-Chang](https://github.com/XiShanYongYe-Chang)
**Fixes:** [#571](https://github.com/kubernetes-sigs/prow/issues/571)
**Status:** MERGED | **Tests:** PASS | **Delta:** +26 / -78

## Context & Motivation

GitHub [no longer sends notifications](https://github.blog/changelog/2025-11-07-removing-notifications-for-mentions-in-commit-messages/) for `@mentions` in commit messages. The original reason for blocking PRs with `@mentions` was to prevent unintended notification spam — that reason no longer applies.

This PR removes the `@mention` check from the `invalidcommitmsg` and `retitle` plugins, retaining only the close-issue keyword check (`fixes #N`, `closes #N`, `resolved #N`, etc.).

Confirmed by [GitHub staff](https://github.com/kubernetes-sigs/prow/issues/571#issuecomment-3805719155).

## Files Changed

| File | What changed | + | - |
|------|-------------|---|---|
| `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go` | Removed `AtMentionRegex`, removed `@mention` checks, updated messages | +7 | -11 |
| `pkg/plugins/invalidcommitmsg/invalidcommitmsg_test.go` | Removed `@mention` test cases, updated expected comment bodies | +16 | -44 |
| `pkg/plugins/retitle/retitle.go` | Removed `AtMentionRegex` check in title validation, updated error | +2 | -2 |
| `pkg/plugins/retitle/retitle_test.go` | Removed `@mention` title test, updated expected message | +1 | -21 |

## Change Walkthrough

### 1. `AtMentionRegex` removal (invalidcommitmsg.go)

The exported `AtMentionRegex` is deleted entirely. `CloseIssueRegex` is kept as a standalone var. No remaining references to `AtMentionRegex` exist in the codebase (verified via grep).

### 2. Commit & title validation (invalidcommitmsg.go)

Both commit message and PR title checks simplified from `CloseIssueRegex.MatchString(...) || AtMentionRegex.MatchString(...)` to just `CloseIssueRegex.MatchString(...)`.

### 3. Retitle plugin (retitle.go)

Same pattern: condition simplified, error message updated from "close issues and at(@) mentions" to "close issues".

### 4. User-facing message updates

| Location | Before | After |
|----------|--------|-------|
| Commit comment body | ...close issues **and at(@) or hashtag(#) mentions** are not allowed | ...close issues **and hashtag(#) mentions** are not allowed |
| Title comment body | ...close issues **and at(@) mentions** are not allowed | ...close issues **are not** allowed |
| Retitle error | ...close issues **and at(@) mentions** | ...close issues |
| Help description | ...contain **@ mentions or** keywords | ...contain keywords |
| Package doc | ...with **@mentions or** keywords | ...with keywords |

### 5. Test updates

- Removed all `@mention` commit fixtures (`sha` indices renumbered accordingly)
- Removed the "contains invalid title with @mention" test case entirely
- Updated the "invalid title and invalid commits" test to use `fixes #9999` as the invalid title
- Updated the "edited with invalid title" test to use `fixes #9999`
- Removed the retitle "@mention to OWNERS" test case
- Updated expected comment strings in retitle tests

## Review Checklist

- [x] **Correctness:** `AtMentionRegex` fully removed, zero dangling references (grep-verified)
- [x] **Tests pass:** Both `invalidcommitmsg` and `retitle` packages pass
- [x] **Test coverage:** Close-issue keyword paths still fully tested; `@mention` test cases properly removed
- [x] **User-facing messages:** All five message strings updated consistently
- [x] **Scope:** Minimal, surgical change — no unrelated modifications
- [x] **No regressions:** The remaining `CloseIssueRegex` logic is untouched
- [x] **Motivation:** Well-documented — linked GitHub changelog and staff confirmation

## Maintainer Perspectives

### Code Quality — APPROVE

- No critical issues. The removal is correct, complete, and leaves no dead code.
- `var` block properly simplified from multi-var to single-var — idiomatic Go.
- Test cases thoroughly updated: fixtures renumbered, expected strings aligned, no stale references.
- The "invalid title and invalid commits" test correctly switches to `fixes #9999` as the trigger.
- No dead imports or unused variables remain.

### Maintainability — APPROVE (Burden: LOW)

- Net reduction in maintenance burden — deletes more code than it adds.
- Removing exported `AtMentionRegex` eliminates the only cross-package coupling point between `invalidcommitmsg` and `retitle` for this feature.
- Test cleanup is thorough: `@mention` scenarios fully removed, not commented out or flagged.
- Change is easily revertible as a single commit if GitHub ever re-introduces mention notifications.

### Deployment Risk — LOW RISK

- No configuration, API, or schema changes. Existing YAML/JSON configs parse identically.
- Behavioral change is **permissive**, not restrictive — PRs with `@mentions` will no longer be blocked.
- `/retitle Add @someone to OWNERS` will now be accepted instead of rejected.
- PRs carrying `do-not-merge/invalid-commit-message` solely due to `@mentions` will have the label auto-removed on next PR event.
- No downtime required. Rollback is safe (just re-adds the stricter check).

## Converging Concerns

**Flagged by 3/3 reviewers:** `invalidCommitMsgCommentBody` (line 38) still says "and hashtag(#) mentions are not allowed in commit messages" but `CloseIssueRegex` only matches close-keywords followed by `#N` (e.g. `fixes #123`), not bare `#123` references. Since the author already edited this line to remove "at(@) or", dropping "and hashtag(#) mentions" would align it with `invalidTitleCommentBody` which already reads: "Keywords which can automatically close issues are not allowed."

**Status:** Non-blocking suggestion. Can be addressed now or in a follow-up.

## Final Recommendation: Approve with Suggestions

All three reviewers unanimously approve. The PR is a clean removal of dead functionality motivated by an upstream GitHub platform change. The only substantive feedback is the misleading "hashtag(#) mentions" wording — a minor, non-blocking improvement that could be addressed in a follow-up.

## Followups

### 1. cleanup: Fix misleading "hashtag(#) mentions" wording in invalidCommitMsgCommentBody

- **Category:** cleanup
- **Necessity:** should
- **Where:** `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:38`
- **Handoff prompt:**

```
In `kubernetes-sigs/prow`, following the merge of PR #607 — "No longer check at mentions in commit messages and PR title" (merge commit 95b2a34128de51a4f618c8d6bb9d0c6b587fd29c), fix a misleading user-facing message left over from that PR.

The constant `invalidCommitMsgCommentBody` in `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go` (around line 38) currently reads:

  "[Keywords](https://help.github.com/articles/closing-issues-using-keywords) which can automatically close issues and hashtag(#) mentions are not allowed in commit messages."

The phrase "and hashtag(#) mentions" is misleading: the plugin's `CloseIssueRegex` only matches close-issue keywords followed by `#N` (e.g. `fixes #123`), not bare `#123` references. The sibling constant `invalidTitleCommentBody` (same file, around line 50) already reads correctly: "which can automatically close issues are not allowed in the title of a Pull Request."

Do the following:
1. Change `invalidCommitMsgCommentBody` to remove "and hashtag(#) mentions", so it reads: "which can automatically close issues are not allowed in commit messages."
2. Update the corresponding test fixture `invalidCommitComment` in `pkg/plugins/invalidcommitmsg/invalidcommitmsg_test.go` to match the new wording.
3. Run `go test ./pkg/plugins/invalidcommitmsg/...` and `go test ./pkg/plugins/retitle/...` and confirm they pass.

Acceptance criteria: the commit message comment body no longer mentions "hashtag(#) mentions", the title comment body is unchanged, and all tests pass.

Out of scope: any logic changes to `CloseIssueRegex`, changes to the retitle plugin, or changes to the GitHub help URL.
```

### 2. cleanup: Update stale help.github.com URL to docs.github.com

- **Category:** cleanup
- **Necessity:** could
- **Where:** `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:38,50` and `pkg/plugins/retitle/retitle.go:143`
- **Handoff prompt:**

```
In `kubernetes-sigs/prow`, following the merge of PR #607 — "No longer check at mentions in commit messages and PR title" (merge commit 95b2a34128de51a4f618c8d6bb9d0c6b587fd29c), update stale GitHub documentation URLs in the invalidcommitmsg and retitle plugins.

Three user-facing bot comment constants reference `https://help.github.com/articles/closing-issues-using-keywords`, which 301-redirects to `https://docs.github.com/articles/closing-issues-using-keywords`. Update them to the current canonical URL:

1. `invalidCommitMsgCommentBody` in `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go` (around line 38)
2. `invalidTitleCommentBody` in the same file (around line 50)
3. The retitle error message in `pkg/plugins/retitle/retitle.go` (around line 143)

Also update the corresponding test fixture strings:
- `invalidCommitComment` and `invalidPRTitleComment` in `pkg/plugins/invalidcommitmsg/invalidcommitmsg_test.go`
- The `expectedComment` strings in `pkg/plugins/retitle/retitle_test.go` that contain this URL

Run `go test ./pkg/plugins/invalidcommitmsg/...` and `go test ./pkg/plugins/retitle/...` and confirm they pass.

Acceptance criteria: all occurrences of `help.github.com` in these two packages are replaced with `docs.github.com`, and all tests pass.

Out of scope: any logic changes, searching for this URL outside these two packages, or changes to any other URLs.
```
