---
pr: kubernetes-sigs/prow#677
title: "Add org invite configuration options"
head_sha: 1f8d0b8ae5238a835eefacc42c19ecd250b24a9b
base: main
reviewed_at: 2026-09-28T17:36:46Z
verdict: approve
---

# Review

## Verdict

Approve with suggestions. The change has low deployment risk and preserves existing behavior when the new settings are absent.

The code quality and deployment reviewers both found that a zero threshold still depends on a successful GitHub search. The maintainability and deployment reviewers both found that the generated configuration reference omits the new settings. These are confirmed, low-impact follow-up items; PR #677 has already merged.

## What this PR does

- Adds `org_invite.prominent` settings to each trigger configuration.
- Lets operators disable the prominent invitation or change its merged PR threshold.
- Lets operators replace the prominent invitation text using a `{join_org_url}` placeholder.
- Preserves the existing invitation text and threshold when the settings are absent.

## Findings

### [should-fix] Handle a zero threshold before searching GitHub
- where: `pkg/plugins/trigger/pull-request.go:343-352`
- concern: The new configuration and test explicitly support `merged_pr_threshold: 0`, which should highlight the invitation for every eligible human author. A search error returns `false` before the zero threshold is checked, suppressing that invitation during an API failure and making an unnecessary request for every such PR. Add a test with the search error client when fixing this.
- excerpt: |
    issues, err := c.GitHubClient.FindIssuesWithOrg(org, query, "", false)
    if err != nil {
        // Search failures should not block the welcome message; fall back to the default copy.
        if c.Logger != nil {
            c.Logger.WithError(err).WithField("author", author.Login).WithField("org", org).Debug("Failed to query merged PRs for join-org guidance")
        }
        return false
    }
    return len(issues) >= cfg.Prominent.EffectiveMergedPRThreshold()

### [should-fix] Regenerate the documented plugin configuration
- where: `pkg/plugins/config.go:549`
- concern: The new `org_invite.prominent` options are absent from the checked-in configuration reference. `hack/gen-prow-documented` generates this file from the configuration structs, and `make update-codegen` runs that generator. Regenerating it would make the available settings and their comments visible to operators.
- excerpt: |
    OrgInvite OrgInviteConfig `json:"org_invite,omitempty"`

## Checked

- Compared the PR head with its parent commit and read issue #670 and the PR description.
- `go test ./pkg/plugins/trigger -run 'TestShouldHighlightJoinOrgMessage|TestOrgInvitationGuidance' -count=1` passed.
- An unset threshold remains 3, and the new test confirms that an explicit threshold of 0 is distinct when search succeeds.
- The deployment review found no breaking configuration or permission changes; search errors still leave the welcome comment intact.
- Repo trigger entries take precedence over org entries as whole configurations, so operators must set `org_invite` on each applicable entry. This selection behavior predates the PR.
- The current codegen verification script does not compare `pkg/plugins`, so it does not catch the missing configuration reference entry.

## Open questions

None.

## Followups

No followups accepted; three candidates were skipped.
