---
pr: kubernetes-sigs/prow#995
title: "branchprotector: add `require_last_push_approval` and `bypass_pull_request_allowances.apps`"
head_sha: f14b640b59d7f9f82dda202cfa904fd58f5fbe99
base: main
reviewed_at: 2026-10-14T15:55:59Z
verdict: approve
gate:
  decision: hold
  gated_at: 2026-10-15T10:49:09Z
  gated_head_sha: f14b640b59d7f9f82dda202cfa904fd58f5fbe99
  reviewed_head_sha: f14b640b59d7f9f82dda202cfa904fd58f5fbe99
---
# Review

## Gate

**Decision: HOLD**

The concern raised by @petr-muller about omitted settings resetting manually managed GitHub policy is addressed by the current preservation helper in `f14b640b59d7f9f82dda202cfa904fd58f5fbe99`. One substantive local review finding remains: the config says an empty app list removes apps, while inherited policies union parent and child app lists. That can leave inherited bypass access in place when an operator expects an empty child list to remove it; clarify the behavior or change the merge semantics before merging.

### Findings disposition

- **Addressed — @petr-muller issue comment (2026-10-10):** the current code preserves omitted app allowances and last-push approval from GitHub state and warns operators to configure them explicitly.
- **Not addressed — `REVIEW.md` finding:** `pkg/config/branch_protection.go:112-116, 215-226` still documents an empty app list as removing apps, but `mergeBypassRestrictions` unions child and parent lists. An empty child list cannot remove apps inherited from a parent.
- No submitted reviews or inline review comments were found. No later response clarifies the inherited-app behavior.

### Gating list

- Clarify the meaning of `apps: []` under policy inheritance or implement removal semantics (`pkg/config/branch_protection.go:112-116, 215-226`). This gates merge because operators may believe they removed inherited app bypass access when the merged policy still allows it.

### Independent merge risk

- The new config fields are optional; existing config files remain valid. The current preservation logic avoids resetting existing GitHub values when these fields are omitted. No other backward-incompatible API, config, or behavioral change was identified in the full PR diff against `main` (`b4b2b38994859131ab14b656b584e80b325096c0`..`f14b640b59d7f9f82dda202cfa904fd58f5fbe99`).
- Preserved enabled/nonempty values produce warnings until operators set them explicitly. This is documented and does not require a coordinated rollout.
- App allowance inheritance is the outstanding configuration risk described in the gating list. No specialized API/config compatibility skill was available; this pass inspected the full diff directly.


## Verdict

Approve with suggestions. The three maintainer perspectives found no blocking issues, and the preservation logic reduces upgrade risk for settings that operators already manage on GitHub. Clarify how app allowances merge through inherited policies and give the temporary compatibility backstop a tracked sunset path.

## What this PR does

- Adds `require_last_push_approval` to branch protection review policy.
- Adds app slugs to pull request bypass allowances.
- Keeps the new settings optional so omitted values differ from explicit false or empty values.
- Preserves current GitHub settings when these new fields are omitted, with warnings to configure them explicitly.
- Adds tests and updates configuration examples and operator documentation.

## Findings

### [should-fix] Clarify inherited app allowance behavior
- where: `pkg/config/branch_protection.go:112-116, 215-226`
- concern: The field comment says an empty app list removes apps, but policy merging unions child and parent lists. An empty child list therefore cannot remove apps inherited from a parent, which may mislead operators configuring policy inheritance. Clarify the documentation or define replacement semantics.
- excerpt: |
    return &BypassRestrictions{
        Apps:  unionStringPtrs(parent.Apps, child.Apps),
        Users: unionStrings(parent.Users, child.Users),
        Teams: unionStrings(parent.Teams, child.Teams),
    }

### [nit] Track the preservation backstop's sunset
- where: `cmd/branchprotector/protect.go:560-565`
- concern: The TODO says to remove this compatibility behavior after configs have had time to set the fields, but gives no timeframe or tracking reference. Tie cleanup to a release or tracked follow-up so the temporary behavior does not become permanent by default.
- excerpt: |
    // TODO: remove this backstop once configs had time to set these fields explicitly,
    // so they behave consistently with all other fields.

## Resolved

### Omitted review settings could reset existing GitHub values

The preservation helper now copies the current app allowances and last-push approval setting into the reconciliation request when configuration omits them. Explicit values remain configurable.

- where: `cmd/branchprotector/protect.go:575-588`
- excerpt: |
    if reviews.BypassRestrictions.Apps == nil && stateReviews != nil && stateReviews.BypassRestrictions != nil && len(stateReviews.BypassRestrictions.Apps) > 0 {
        apps := make([]string, 0, len(stateReviews.BypassRestrictions.Apps))
        for _, app := range stateReviews.BypassRestrictions.Apps {
            apps = append(apps, app.Slug)
        }
        reviews.BypassRestrictions.Apps = &apps
    }

## Checked

- Reviewed the PR diff at head `f14b640b59d7f9f82dda202cfa904fd58f5fbe99` against `main` from code quality, maintainability, and deployment risk perspectives.
- Confirmed omitted review settings are preserved before reconciliation comparison; explicit values can be managed through config.
- Confirmed tests cover preservation and explicit overrides, and existing configuration remains compatible.
- No tests were run as part of this review.

## Open questions

None.
