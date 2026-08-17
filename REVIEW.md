---
pr: kubernetes-sigs/prow#755
title: "assign: add only_org_members config with warn/block action"
head_sha: 88366d45c02a0429dd163a8c8fe4e68697a6603c
base: main
reviewed_at: 2026-08-17T22:39:03Z
verdict: request-changes
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
  - old_sha: 88366d45c02a0429dd163a8c8fe4e68697a6603c
    new_sha: 88366d45c02a0429dd163a8c8fe4e68697a6603c
    summary: "No code changes. Prucek reapplied /lgtm on 2026-08-11. petr-muller submitted a CHANGES_REQUESTED review on 2026-08-12 raising a new should-fix (Assign config struct conflates plugin identity with the restriction feature; proposes nesting OnlyOrgMembers/Action/ExemptLabels under a Restrict-style struct before merge) plus three open questions (PR vs issue scope, org-level key support, self/someone-else assign semantics). Followed by a same-day clarifying comment softening tone, not withdrawing the request."
---

## Findings

### [should-fix] HasConfigFor() not updated for Assign
- where: `pkg/plugins/config.go:2467-2559`
- concern: |
    `HasConfigFor()` is consumed only by `cmd/checkconfig`'s supplemental-config-layout
    validator (`cmd/checkconfig/main.go:1374-1420`), which enforces that a config file
    under `myorg/` (or `myorg/myrepo/`) — the standard sharded-`plugins.yaml` layout large
    Prow deployments use — contains only config scoped to that org/repo. `Assign` is not
    part of the function's equality baseline or its `orgs`/`repos` enumeration, so any file
    that sets `assign:` makes `reflect.DeepEqual` fail and forces `global = true`
    unconditionally. Concretely, a file at `myorg/plugins.yaml` containing only
    `assign: {myorg: {only_org_members: true}}` is correctly scoped but would be rejected
    by `checkconfig` with "contains global config" — a false positive. Not silent data
    loss (runtime config loading is unaffected), but a real, reachable break of the
    verification path this feature is meant to be used through. `TestHasConfigFor`
    (`pkg/plugins/config_test.go:2051`) has no case for `Assign`. Raised again explicitly
    in petr-muller's 2026-08-12 CHANGES_REQUESTED review.
- excerpt: |
    // Missing: iteration over c.Assign map keys

### [should-fix] Config struct conflates plugin identity with the restriction feature
- where: `pkg/plugins/config.go:115-128`
- concern: |
    `OnlyOrgMembers`, `Action`, and `ExemptLabels` are all coupled knobs of one restriction
    feature, but live flat on `Assign` (named after the plugin, not the feature). Two of
    the three fields (`Action`, `ExemptLabels`) only have meaning when `OnlyOrgMembers` is
    `true` — nothing in `validateAssign()` enforces that pairing, so
    `action: block` with `only_org_members` unset validates but silently does nothing
    (verified: the gate at `assign.go:48`/`assign.go:216` is `if cfg.OnlyOrgMembers`, `Action`
    is never consulted otherwise). Since config schema is a de facto stable API once
    released (every adopting org writes YAML against it; renaming shipped fields requires a
    deprecation cycle), this is the last free opportunity to fix the shape — `Assign` isn't
    on `main` yet. Proposed by petr-muller (2026-08-12 review): nest the three fields under
    a `Restrict` sub-struct (e.g. `assign.<org/repo>.restrict.{action,exempt_labels}`),
    using the sub-struct's presence (non-nil) as the enablement signal instead of the
    `OnlyOrgMembers` bool. This also leaves room for a future non-org-membership
    restriction mechanism without another top-level bool/struct.
- excerpt: |
    type Assign struct {
        OnlyOrgMembers bool         `json:"only_org_members,omitempty"`
        ExemptLabels   []string     `json:"exempt_labels,omitempty"`
        Action         AssignAction `json:"action,omitempty"`
    }

### [question] Restriction scope and semantics are unclear
- where: `pkg/plugins/assign/assign.go:94-104`, `assign.go:216-225`
- concern: |
    (1) PR vs issue: `handleGenericComment` runs the `/assign` restriction path
    unconditionally — there is no `e.IsPR` gate (only the separate `/cc` review-request
    handler at line 101 is PR-gated). So `only_org_members` currently applies identically
    to `/assign` on PRs and issues; the warn message is also hardcoded to say "issue"
    (`assign.go:262`) even when triggered from a PR comment. (2) Member/non-member
    direction: `orgMemberAssignFunc`/`splitByMembership` check membership of the
    **assignment target**, not the commenter — so a non-member can freely `/assign` an
    org-member (e.g. an approver, per the standard OWNERS approval-request flow), while an
    org member attempting to assign a non-member is blocked/warned identically to a
    non-member self-assigning. Docs don't currently spell this out. Raised by petr-muller's
    2026-08-12 review as an open question about intended scope; needs an explicit answer
    (and doc wording) from the author, not necessarily a code change if current behavior is
    intended.

### [question] Org-level (bare org, not org/repo) key not explicitly covered by tests
- where: `pkg/plugins/assign/assign_test.go:472-597`
- concern: |
    `AssignFor()` (`config.go:1111-1122`) resolves `org/repo` → `org` → `"*"` → zero-value
    with no special-casing, so a bare-org key mechanically works the same as an org/repo
    key. Not verified whether the test suite exercises the org-only key specifically (only
    org/repo-keyed cases were seen in the reviewed range). Raised by petr-muller's
    2026-08-12 review; needs confirmation/added coverage rather than a design change.

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
- Prucek gave /lgtm on 2026-06-24 and a formal `APPROVED` review on 2026-06-26; the lgtm label was auto-removed on 2026-07-09 when the branch was rebased onto main (prow-bot policy: any new commit strips lgtm, even a rebase with no functional delta) and was reapplied via `/lgtm` on 2026-08-11
- PR currently awaits approval from an OWNERS approver (`cjwagner`) for `pkg/plugins/OWNERS` — unrelated to open review findings
- petr-muller submitted a formal `CHANGES_REQUESTED` review on 2026-08-12T15:11:45Z (content folded into Findings above as two new should-fix/question items) and a same-day follow-up comment (2026-08-12T15:18:48Z) clarifying tone was not intended harshly and the feature is valued — but not withdrawing the change request; the config-struct-shape concern is explicitly maintained as the primary reason for extra scrutiny

## Open questions
- Is there an intentional reason `HasConfigFor()` was not updated, or was it missed?
- In warn mode, should the comment be suppressed when `AssignIssue` itself fails for the non-member?
- Should `Assign`'s restriction fields be nested under a dedicated `Restrict` struct before this merges? (see should-fix above)
- Is `only_org_members` intended to apply to PRs as well as issues? If so, the warn message wording needs fixing too.
- Can a member assign a non-member and vice versa — is the current target-membership-only check the intended semantics?
