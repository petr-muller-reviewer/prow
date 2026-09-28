---
pr: kubernetes-sigs/prow#977
title: "`peribolos`: exclude enterprise-team members from `--dump`; fail-loud on enterprise listing errors"
head_sha: 5800d630501be653dd629903bd3ed65fffbd0f65
base: main
reviewed_at: 2026-09-27T11:50:17Z
verdict: approve
refresh_log:
  - old_sha: dc32fb065cb63956e0f62b0e75adae4cebfd202d
    new_sha: 5800d630501be653dd629903bd3ed65fffbd0f65
    summary: "Reviewed CLI help, helper comment, and error-message wording; hold was lifted."
---

# Review

## Verdict

Approve with a non-blocking documentation suggestion.

The filtering predicate, safety checks, and client representation match the stated behavior. Both dump and apply stop before using an incomplete enterprise-member set, avoiding silent drift or removals; the dump path additionally refuses an unreported `direct_membership` value rather than guessing. The intentional fail-loud behavior creates an operational prerequisite worth documenting for installations that enable this flag.

## What this PR does

- Adds a client method and nullable representation for GitHub's `direct_membership` field.
- Builds one normalized set of enterprise-team members and treats errors while building it as fatal.
- Makes `--dump --ignore-enterprise-teams` omit only indirect-only enterprise members that are not emitted in a regular team.
- Retains the unfiltered admin list for validating the dump token and avoids serializing the read-only field on membership updates.
- Adds coverage for indirect/direct membership, missing fields and errors, case normalization, the regular-team round trip, and client decoding.

Since previous review:

- Clarifies the `--ignore-enterprise-teams` help text and the shared helper's description of apply and dump behavior.
- Removes an inaccurate enterprise-team permission hint from the org-membership lookup error.

## Findings

### [should-fix] Document the fail-loud operational prerequisites
- where: `cmd/peribolos/main.go:568-580`
- concern: Existing `--ignore-enterprise-teams` users now stop the complete per-org reconciliation when any enterprise team's members cannot be listed. The release note and CLI help describe the behavior, but operator-facing documentation should state the required token access and that a failure skips subsequent work for that org; dump users on GHES also need an API version that supplies `direct_membership`.
- excerpt: |
    enterpriseMembers, err := enterpriseTeamMembers(client, orgName, allTeams)
    if err != nil {
        return fmt.Errorf("failed to list %s enterprise team members (does the token have org admin/read access to enterprise team membership?): %w", orgName, err)
    }

## Checked

- `enterpriseTeamMembers` aggregates listing failures and both callers stop before using a partial result.
- The regular-team guard records maintainers and members before the org-level filtering, preserving the config invariant for emitted teams.
- `DirectMembership` is a pointer with `omitempty`, while `UpdateOrgMembership` constructs a fresh request body.
- The new API calls and failure modes apply only when `--ignore-enterprise-teams` is enabled.
- Reviewed the wording-only follow-up commit `5800d630501be653dd629903bd3ed65fffbd0f65` against the prior review point.
- `git diff --check f21dfc59dd24d5f364a427f5d1a9d13ce4d3e598...HEAD`.
- `go test ./cmd/peribolos ./pkg/github`.

## Open questions

None.
