---
pr: kubernetes-sigs/prow#752
title: ".*: add `delete-spam-issue` prow plugin"
head_sha: adac22030a7c19f4bab2b2f6806c5f353d7f436e
base: main
reviewed_at: 2026-06-12T15:59:05Z
verdict: approve
---

## Summary

- New prow plugin: `/delete-spam-issue` command deletes GitHub issues via GraphQL `deleteIssue` mutation.
- Authorization via three mechanisms: explicit user allowlist, GitHub team membership, top-level OWNERS approvers (on by default, configurable).
- Config struct `DeleteSpamIssue` added to plugin config with `allowed_users`, `allowed_teams`, `allow_top_level_approvers`.
- Rejects use on PRs. Posts audit comment before deleting. Posts error comment on failure.
- Second commit adds the `allow_top_level_approvers` config knob (default true).

## Findings

### [should-fix] No log entry on successful deletion
- where: `pkg/plugins/delete-spam-issue/delete-spam-issue.go:114-119`
- concern: When deletion succeeds, the handler returns nil with no log output. The pre-deletion comment is destroyed along with the issue, so the only durable audit trail is email notifications (if they fired in time). A structured log line recording who deleted which issue would be the only reliable record. Flagged independently by Code Quality, Maintainability, and Deployment Risk reviewers.
- excerpt: |
    if err := deleteIssue(gc, org, e.NodeID); err != nil {
        log.WithError(err).Error("Failed to delete issue.")
        return gc.CreateComment(...)
    }
    return nil

### [should-fix] Rejection message misleading when not all auth paths configured
- where: `pkg/plugins/delete-spam-issue/delete-spam-issue.go:149-155`
- concern: The rejection message always says "you are not in one of the allowed teams and are not an allowed user" even when no teams are configured (only AllowedUsers or only OWNERS). The message should be built dynamically based on which authorization sources are actually present. Flagged by Code Quality and Maintainability reviewers.
- excerpt: |
    msg := "This issue cannot be deleted, because you are not in one of the allowed teams and are not an allowed user."
    if len(config.AllowedTeams) > 0 {
        msg += fmt.Sprintf(" Must be a member of one of these teams: %s", strings.Join(config.AllowedTeams, ", "))
    }

### [should-fix] Documented default for allow_top_level_approvers is misleading
- where: `pkg/plugins/plugin-config-documented.yaml:412`
- concern: The generated YAML shows `allow_top_level_approvers: false` (Go zero value) but the runtime default is `true` via the `*bool` + `TopLevelApproversAllowed()` method. For a security-sensitive field controlling who can permanently delete issues, the mismatch between documented value and runtime default could mislead operators into thinking OWNERS approvers are excluded when they are included by default.

### [nit] canNotDeleteReason naming
- where: `pkg/plugins/delete-spam-issue/delete-spam-issue.go:122`
- concern: `canNotDeleteReason` reads awkwardly. `denyReason` or `denialReason` is more conventional Go style.
- excerpt: |
    func canUserDeleteIssue(...) (canDelete bool, canNotDeleteReason string, err error) {

### [nit] Unrelated invalid_commit_msg in documented config
- where: `pkg/plugins/plugin-config-documented.yaml:459-469`
- concern: The regenerated documented config picks up the `invalid_commit_msg` section from the recently merged PR #738. Not a problem (it is correct), but it is out-of-scope noise in this diff.

### [question] Standalone plugin vs issue-management
- where: `pkg/plugins/delete-spam-issue/`
- concern: The existing `issue-management` plugin already handles `/pin-issue`, `/unpin-issue`, `/link-issue`, `/unlink-issue` with the same `githubClient` and `ownersClient` interfaces. This could be another command there. The separate plugin is defensible given the distinct auth model and destructive nature, but worth an explicit decision from maintainers.

### [question] GitHub App permission requirements
- where: `pkg/plugins/delete-spam-issue/delete-spam-issue.go:174-179`
- concern: The `deleteIssue` GraphQL mutation requires admin-level repository access. The PR does not document this requirement. Operators enabling the plugin need to know what GitHub App permissions to grant. Flagged by Deployment Risk reviewer.

### [question] Global config vs per-org/repo config
- where: `pkg/plugins/config.go:80`
- concern: The `DeleteSpamIssue` config is a single global struct, not a per-org/repo map. All repos that enable this plugin share the same `allowed_users`, `allowed_teams`, and `allow_top_level_approvers` settings. Multi-tenant Prow installations wanting different authorization rules per org/repo cannot achieve that with this design. Not blocking for initial merge but worth flagging as a design choice.

## Checked

- `GenericCommentEvent.NodeID` is the issue's node ID (not the comment's) for both `IssueEvent` and `IssueCommentEvent` paths in `pkg/github/helpers.go:126-168`. The `IsPR` guard prevents PR/review event paths from reaching delete logic.
- `DeleteIssueInput` exists in the vendored `shurcooL/githubv4@v0.0.0-20210725200734` at `input.go:832`.
- `MutateWithGitHubAppsSupport` signature matches the interface declaration.
- Authorization checks: user allowlist is case-insensitive via `strings.EqualFold`, team check delegates to GitHub API, OWNERS check uses `github.NormLogin`.
- Test coverage: allowed user, team member, top-level approver, case-insensitive match, PR rejection, non-matching body, non-created action, LoadRepoOwners failure, team API failure, mutation failure, disabled allow_top_level_approvers (14 test cases).
- Plugin registration follows existing patterns (init + RegisterGenericCommentHandler).
- Both `cmd/hook` and `pkg/hook` plugin-imports updated.
- `helpProvider` generates correct yaml snippet and command help.
- `context.Background()` in the GraphQL call is consistent with other Prow plugins.
- `fakeOwnersClient` duplication with `pin-issue_test.go` is acknowledged via TODO.

## Open questions

- Should this live in `issue-management` instead of as a standalone plugin? The auth model is different (explicit allowlists vs org membership/OWNERS), but it is conceptually another issue management command.
- Does the Prow GitHub App in kubernetes org already have the permissions needed for the `deleteIssue` GraphQL mutation (requires admin-level access)?
- Is global config intentional, or should per-org/repo authorization be supported from the start?
