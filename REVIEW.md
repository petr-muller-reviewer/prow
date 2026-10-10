---
pr: kubernetes-sigs/prow#995
title: "branchprotector: add `require_last_push_approval` and `bypass_pull_request_allowances.apps`"
head_sha: 5ddea54f47dbf78bbfcf31cf8c7bb6f108e34423
base: main
reviewed_at: 2026-10-09T10:40:23Z
verdict: approve
gate:
  decision: hold
  gated_at: 2026-10-10T15:22:53Z
  gated_head_sha: 5ddea54f47dbf78bbfcf31cf8c7bb6f108e34423
  reviewed_head_sha: 5ddea54f47dbf78bbfcf31cf8c7bb6f108e34423
---
# Review

## Gate

**Decision: HOLD**

The PR head is unchanged since the review. The unresolved question about omitted app bypass allowances affects existing deployments that configure user or team bypasses in Prow while managing app allowances elsewhere. Hold for the author to confirm the intended behavior; if omission clears existing allowances, document the migration so operators can configure them before rollout.

### Findings disposition

- **Can't tell — omitted app allowances** (`REVIEW.md`, `cmd/branchprotector/request.go:97-103`): the current code always sends `apps: []` when the config omits `apps`; `equalBypassRestrictions` detects existing app allowances as a mismatch and triggers reconciliation. Whether GitHub clears that app list and whether omission is intended to clear it need confirmation. Recommended disposition: preserve omitted allowances, or document that Prow takes ownership of the full list and require operators to migrate existing values.
- The policy merge test suggestion remains non-gating.
- No submitted reviews or inline review comments were found. The author's October 8 issue comment asks whether adding the app field to this PR is acceptable; no maintainer response is present.

### Independent merge risk

- **App bypass config** (`cmd/branchprotector/request.go:97-103`): existing branches with `bypass_pull_request_allowances` configured for users or teams and app allowances maintained outside Prow may have those app allowances removed on reconciliation. This affects each such branchprotector-managed branch; the API documents replacement for user/team arrays but does not directly spell out the app-array case. The PR adds a config example but no upgrade note.
- **Last-push approval default** (`cmd/branchprotector/request.go:147`, `cmd/branchprotector/protect.go:658`): an omitted setting becomes `false` and is now compared against GitHub state. Branches with review policy managed by Prow and `require_last_push_approval` enabled outside Prow will be reset to false on reconciliation. Operators must set it to true in Prow config to retain it; the documented YAML shows the default but gives no upgrade guidance.
- Both new config fields are optional, so existing config files continue to parse. No removed or renamed API fields or new credential requirements were found.

### Gating list

- Confirm whether omitting `bypass_pull_request_allowances.apps` should preserve or clear existing GitHub app allowances. If clearing is intentional, document the migration and release impact before merge.

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
