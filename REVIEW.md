---
pr: kubernetes-sigs/prow#995
title: "branchprotector: add `require_last_push_approval` and `bypass_pull_request_allowances.apps`"
head_sha: 5ddea54f47dbf78bbfcf31cf8c7bb6f108e34423
base: main
reviewed_at: 2026-10-09T10:40:23Z
verdict: approve
---
# Review

## Verdict

Approve with suggestions. The new policy fields are wired through configuration, merging, GitHub request/response types, comparisons, and documentation. Please clarify whether omitting `apps` should preserve existing app bypass allowances; the code sends an empty list, but the replacement behavior for apps is not directly stated in the API docs.

## What this PR does

- Adds `require_last_push_approval` to branch protection review policy.
- Adds app slugs to pull request bypass allowances.
- Includes both fields in inherited policy and GitHub API comparisons.
- Updates configuration examples and branchprotector documentation.

## Findings

### [question] Clarify whether omitted app allowances are preserved
- where: `cmd/branchprotector/request.go:97-103`
- concern: When bypass restrictions are configured without `apps`, this constructs an explicit empty app list. Existing app allowances then compare as a mismatch, so reconciliation may clear them. Should omission preserve existing app allowances? If so, keep the request field nil when config omits it, matching the existing `equalApps` compatibility behavior.
- excerpt: |
    apps := append([]string{}, sets.List(sets.New[string](rp.Apps...))...)
    teams := append([]string{}, sets.List(sets.New[string](rp.Teams...))...)
    users := append([]string{}, sets.List(sets.New[string](rp.Users...))...)
    return &github.BypassRestrictionsRequest{
        Apps:  &apps,
        Teams: &teams,
        Users: &users,
    }

### [nit] Add direct tests for policy merge behavior
- where: `pkg/config/branch_protection.go:175-181, 205-209`
- concern: The new boolean override and app-list union use existing merge patterns, but focused tests would protect their inheritance behavior from regressions.
- excerpt: |
    return &ReviewPolicy{
        DismissalRestrictions:   mergeDismissalRestrictions(parent.DismissalRestrictions, child.DismissalRestrictions),
        DismissStale:            selectBool(parent.DismissStale, child.DismissStale),
        RequireOwners:           selectBool(parent.RequireOwners, child.RequireOwners),
        Approvals:               selectInt(parent.Approvals, child.Approvals),
        RequireLastPushApproval: selectBool(parent.RequireLastPushApproval, child.RequireLastPushApproval),
        BypassRestrictions:      mergeBypassRestrictions(parent.BypassRestrictions, child.BypassRestrictions),
    }

## Checked

- Confirmed both new fields flow through config parsing, policy merging, API models, request construction, state comparison, and documentation.
- Confirmed an omitted `require_last_push_approval` maps to `false`; declare it as `true` in Prow config if branchprotector should keep it enabled.
- Existing config files remain valid with the new optional fields.

## Open questions

- When `bypass_pull_request_allowances.apps` is omitted, should branchprotector preserve app bypass allowances already set in GitHub, or treat omission as an instruction to clear them?
