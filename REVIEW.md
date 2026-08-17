---
pr: kubernetes-sigs/prow#755
title: "assign: add only_org_members config with warn/block action"
head_sha: 88366d45c02a0429dd163a8c8fe4e68697a6603c
base: main
reviewed_at: 2026-07-26T22:40:25Z
verdict: approve-with-suggestions
refresh_log:
  - old_sha: 8ce46a93cdbe95b464b0b8ff184584e797b1a55e
    new_sha: aef24cbf4140d92340758f86ad030d10ef010046
    summary: "Config refactored from []Assign with Repos field to map[string]*Assign keyed by org or org/repo (Dco pattern). AssignFor() returns *Assign with repo > org > * fallback. Reviewer Prucek requested Action validation and closure extraction — aligns with our findings."
  - old_sha: aef24cbf4140d92340758f86ad030d10ef010046
    new_sha: 47513e5dbd5e10189c2d059f2abf7c16613db608
    summary: "Author addressed all three should-fix items: AssignAction type + validateAssign() added, closure extracted to orgMemberAssignFunc/splitByMembership/blockNonMembers, nil check removed. Prucek gave /lgtm on 2026-06-24."
  - old_sha: 47513e5dbd5e10189c2d059f2abf7c16613db608
    new_sha: 88366d45c02a0429dd163a8c8fe4e68697a6603c
    summary: "Branch rebased onto latest main (brings in ~50 unrelated files from merged PRs, no functional overlap with assign). Only real content change: one-line clarifying comment added at assign.go:255 addressing Prucek's warn-path request. LGTM label was auto-removed on rebase (2026-07-09) and PR now awaits OWNERS approval; author ran unrelated /remove-area label cleanup."
---

## Findings

### [should-fix] HasConfigFor() not updated for Assign
- where: `pkg/plugins/config.go:2447-2537`
- concern: `HasConfigFor()` enumerates all repo-scoped configs for config sharding. `Assign` (`map[string]*Assign`) is still not included. Config entries could be silently dropped during sharding.
- excerpt: |
    // Missing: iteration over c.Assign map keys

### [nit] Warn message hardcodes "good first issue"
- where: `pkg/plugins/assign/assign.go:256-260`
- concern: The warn message tells users to look for issues labeled `good first issue`, but `ExemptLabels` is configurable and may not include that label.
- excerpt: |
    "If you are new to this project, consider looking for issues labeled "+
        "`good first issue` to get started.",

### [nit] Warn comment posted even when AssignIssue fails
- where: `pkg/plugins/assign/assign.go:251-265`
- concern: In warn mode, `AssignIssue` is called for all logins including non-members, then the warning is posted regardless of whether the assign succeeded. Double-comment possible.
- excerpt: |
    assignErr := gc.AssignIssue(owner, repo, number, logins)
    if len(nonMembers) > 0 {
        warnMsg := fmt.Sprintf(...)

### [nit] Block mode MissingUsers merging conflates two failure modes
- where: `pkg/plugins/assign/assign.go:282-296` (now in `blockNonMembers`)
- concern: When block mode partially assigns members and GitHub rejects some, error merging makes "not an org member" indistinguishable from "GitHub rejected this org member." Now isolated in `blockNonMembers()` which improves readability, but the logic is unchanged.
- excerpt: |
    if merr, ok := assignErr.(github.MissingUsers); ok {
        mu.Users = append(mu.Users, merr.Users...)
    } else if assignErr != nil {
        return assignErr
    }

### [nit] Test coverage gaps
- where: `pkg/plugins/assign/assign_test.go:472-597`
- concern: No test for warn mode with mixed members/non-members. No test for case-insensitive exempt label matching. No tests for the new map-based `AssignFor` lookup (especially the `*` wildcard fallback). Tests now use typed `plugins.AssignActionBlock` constants (good), but no test for `validateAssign` rejecting invalid values.

### [question] Block mode error merging edge case
- where: `pkg/plugins/assign/assign.go:282-294` (now in `blockNonMembers`)
- concern: If `AssignIssue` for members returns a non-`MissingUsers` error and there are also non-members, the real error is returned and the non-members are silently ignored.
- excerpt: |
    } else if assignErr != nil {
        return assignErr
    }

## Resolved

### [previously nit] Prucek requests comment on warn path
- resolved by: Clarifying comment added at `assign.go:255` — `// AssignActionWarn (default): assign everyone but post a nudge comment for non-members.`

### [previously should-fix] Action field is not validated
- resolved by: `AssignAction` type introduced with `AssignActionWarn` and `AssignActionBlock` constants. `validateAssign()` function added and wired into `Validate()` at `config.go:1714`. Tests updated to use typed constants.

### [previously should-fix] Closure in newAssignHandler concentrates complexity
- resolved by: Closure extracted into three named functions: `orgMemberAssignFunc` (returns the closure), `splitByMembership` (separates logins into members/non-members), `blockNonMembers` (handles block action logic). Individually testable and clearer in stack traces.

### [previously nit] Unnecessary nil check in newAssignHandler
- resolved by: Removed. `AssignFor()` always returns non-nil.

### [previously should-fix] Assign does not implement getRepos() / validateRepoDupes
- resolved by: Config refactored from `[]Assign` with `Repos` field to `map[string]*Assign` — map keys naturally prevent duplicate repo entries.

## Checked
- Config struct uses `map[string]*Assign` keyed by org or org/repo (matches `Dco` pattern)
- `AssignFor()` does repo > org > `*` fallback, returns `*Assign` (always non-nil)
- `AssignAction` type with `AssignActionWarn` / `AssignActionBlock` constants
- `validateAssign()` rejects unknown action values at config load time
- Logic extracted to `orgMemberAssignFunc`, `splitByMembership`, `blockNonMembers` — individually testable
- `helpProvider` correctly generates per-repo config info using typed constants
- `githubClient` interface extended cleanly with `GetIssueLabels` and `IsMember`
- `hasExemptLabel` uses case-insensitive comparison
- Zero-value `&Assign{}` preserves backwards-compatible behavior
- Test coverage: 8 table-driven cases for block, warn, mixed, exempt labels, error propagation
- `newReviewHandler` intentionally not affected by this config
- Deployment risk is LOW: all fields optional with `omitempty`, no breaking changes, safe rollback
- `/cc` (review handler) unaffected — blast radius limited to assign handler
- Prucek gave /lgtm on 2026-06-24 and a formal `APPROVED` review on 2026-06-26; the lgtm label was auto-removed on 2026-07-09 when the branch was rebased onto main (prow-bot policy: any new commit strips lgtm, even a rebase with no functional delta) and has not been reapplied
- PR currently awaits approval from an OWNERS approver (`cjwagner`) for `pkg/plugins/OWNERS` — unrelated to open review findings

## Open questions
- Is there an intentional reason `HasConfigFor()` was not updated, or was it missed?
- In warn mode, should the comment be suppressed when `AssignIssue` itself fails for the non-member?
