---
pr: kubernetes-sigs/prow#755
title: "assign: optionally restrict /assign on issues to org members"
head_sha: 1ee75d56f07c27eec50c0714aa2b2762bbdbb472
base: main
reviewed_at: 2026-09-30T15:38:20Z
verdict: request-changes
---

# Review

## Verdict

Request changes.

The opt-in, issue-only policy preserves existing behavior, but three confirmed defects merit changes before merge: malformed scope keys silently never apply, a null repo entry fails to override an org restriction, and warn mode posts a nudge even when GitHub rejects the assignment. Restricted issue assignments also depend on GitHub membership access, so operators should verify the bot's credentials before broad rollout.

## What this PR does

- Adds per-repository, per-organization, and wildcard `assign` configuration with an optional `restrict` block and required `warn` or `block` action.
- Applies the restriction to `/assign` on issues based on the commenter's organization membership; pull requests and `/unassign` remain unrestricted.
- Allows issues with configured exempt labels to bypass the restriction, with case-insensitive label matching.
- Adds configuration validation, supplemental-config merging, scope detection, help text, and tests for the new behavior.

## Findings

### [blocking] Validate assign scope keys
- where: `pkg/plugins/config.go:1685-1690`
- concern: |
    `validateAssign()` checks the action but never validates map keys. A key such as `org/repo/extra` or `""` loads successfully, yet `AssignFor()` only looks up exact `org/repo`, `org`, and `*` keys, so an intended access restriction silently never applies. Reject invalid keys and test the rejected forms.
- excerpt: |
    func validateAssign(assign map[string]*Assign) error {
        for key, cfg := range assign {
            if cfg == nil || cfg.Restrict == nil {
                continue
            }
            switch cfg.Restrict.Action {

### [blocking] Honor a null repo entry as an override
- where: `pkg/plugins/config.go:1176-1184`
- concern: |
    The documented most-specific entry is supposed to replace an org or wildcard restriction, including an empty repo entry used to opt out. YAML `org/repo:` decodes to a present map key with a nil `*Assign`, but `AssignFor()` tests the value for non-nil and falls through to the org entry, so that repo remains restricted. The test covers `org/repo: {}` but not the equally natural null form. Check key presence before falling back, or reject null entries explicitly during validation.
- excerpt: |
    if c.Assign[fmt.Sprintf("%s/%s", org, repo)] != nil {
        return c.Assign[fmt.Sprintf("%s/%s", org, repo)]
    }
    if c.Assign[org] != nil {
        return c.Assign[org]
    }

### [blocking] Suppress the warn nudge when assignment fails
- where: `pkg/plugins/assign/assign.go:264-268`
- concern: |
    For a non-member in warn mode, `AssignIssue` can return `github.MissingUsers` when GitHub rejects an assignee. The code posts the nudge anyway, then returns the error to `handle()`, which posts a second assignment-failure comment. This leaves two bot replies for one command and suggests the assignment proceeded even when it did not. Post the nudge only for a successful assignment, or coordinate the two responses in one place.
- excerpt: |
    assignErr := gc.AssignIssue(owner, repo, number, logins)
    if err := gc.CreateComment(owner, repo, number, plugins.FormatResponseRaw(e.Body, e.HTMLURL, e.User.Login, warnMessage(org, restrict.ExemptLabels))); err != nil {
        log.WithError(err).Error("failed to post warning comment for /assign by a non-org-member")
    }
    return assignErr

### [should-fix] Let an org member assign when label lookup fails
- where: `pkg/plugins/assign/assign.go:275-285`
- concern: |
    With `exempt_labels` configured, `commenterMayAssign()` fetches labels before checking the commenter. A label API failure therefore stops even a confirmed org member's `/assign`, although member assignments do not depend on an exemption. Check membership on label lookup failure so members can still assign while non-members remain fail-closed.
- excerpt: |
    labels, err := gc.GetIssueLabels(owner, repo, number)
    if err != nil {
        return false, err
    }
    if hasExemptLabel(labels, exemptLabels) {
        return true, nil
    }

### [nit] Report a failed warning-comment post
- where: `pkg/plugins/assign/assign.go:265-268`
- concern: |
    When assignment succeeds but `CreateComment` fails, the handler returns success after logging the failure, so users never receive the configured warning. Surface the comment failure to operators without blindly retrying the already completed assignment.
- excerpt: |
    if err := gc.CreateComment(owner, repo, number, plugins.FormatResponseRaw(e.Body, e.HTMLURL, e.User.Login, warnMessage(org, restrict.ExemptLabels))); err != nil {
        log.WithError(err).Error("failed to post warning comment for /assign by a non-org-member")
    }
    return assignErr

### [nit] Add context to lookup errors
- where: `pkg/plugins/assign/assign.go:275-285`
- concern: |
    Raw label and membership API errors are returned without the issue, org, or lookup operation, making failures harder to diagnose in a multi-repository installation. Wrap them with that context.
- excerpt: |
    if err != nil {
        return false, err
    }
    return gc.IsMember(org, commenter)

## Resolved

### [previously should-fix] HasConfigFor() did not recognize Assign
- resolved by: `HasConfigFor()` now includes `Assign` in its equality baseline and enumerates org, repo, and wildcard keys; `TestHasConfigFor` covers all three scopes.

### [previously should-fix] Restriction fields were flat on Assign
- resolved by: The fields now live under optional `Assign.Restrict`, and `validateAssign()` requires `restrict.action` to be `warn` or `block`.

### [previously question] Restriction scope and membership direction were unclear
- resolved by: The code, config comments, help text, and tests now specify issue-only behavior and check the commenter's membership rather than assignment targets.

### [previously question] Org-level and wildcard lookup lacked direct tests
- resolved by: `TestAssignFor` now exercises org, repo, wildcard, precedence, and an empty `{}` repo override.

## Checked

- Reviewed the five-file PR diff against merge base `e62c14122b7de539fb60236fb8085c0fd1f5dc83` at head `1ee75d56f07c27eec50c0714aa2b2762bbdbb472`.
- `newAssignHandler` gates the restriction on `!e.IsPR`; the existing `handle()` uses `gc.UnassignIssue` separately for `/unassign`.
- `commenterMayAssign` bypasses membership checks on exempt labels and propagates label and membership lookup errors.
- `validateAssign` rejects missing or unsupported actions for non-nil `restrict` blocks.
- Supplemental config merging and `HasConfigFor()` include the new `Assign` map.
- Inspected `TestAssignRestrict`, `TestValidateAssign`, `TestAssignFor`, `TestHasConfigFor`, and `TestMergeFrom`; the three blocking scenarios above lack direct coverage. The Code Quality reviewer ran `go test ./pkg/plugins/assign ./pkg/plugins` successfully at this head.
- Existing configurations retain their `/assign` behavior when `restrict` is absent; PR assignments and `/unassign` remain available when it is present.
- GitHub's [organization membership endpoint](https://docs.github.com/en/rest/orgs/members?apiVersion=2022-11-28#check-organization-membership-for-a-user) documents `Members: read` for fine-grained tokens and a `302` response when the requester is not an org member; the Prow client converts that `302` to an error.

## Open questions

- Should an empty repo override written as `org/repo:` disable the inherited restriction, like `org/repo: {}`?
- In warn mode, should a failed assignment produce only the existing GitHub failure response?
- Can the deployment bot read private org membership in each org that will enable this restriction, and will the rollout start with one repository before org-wide or wildcard scope?
