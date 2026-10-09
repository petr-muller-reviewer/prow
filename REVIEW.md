---
pr: kubernetes-sigs/prow#1002
title: "plugin/label: Add OWNERS access and label reset"
head_sha: 64bcfe0914815974fa49624bd107fcd013da59f4
base: main
reviewed_at: 2026-10-09T14:02:19Z
verdict: request-changes
---
# Review

## Verdict

Request changes. The implementation matches the intended opt-in behavior, but the OWNERS authorization check can treat a partial changed-file list as complete on PRs exceeding GitHub's 3,000-file response cap. Because access is supposed to require approval authority for every changed file, the check must fail closed when completeness cannot be established.

## What this PR does

- Adds opt-in OWNERS approver access for restricted labels.
- Adds opt-in removal of configured labels when a PR receives new commits.
- Merges both options across global, organization, and repository configuration scopes.
- Updates plugin help and adds tests for authorization, reset behavior, and config merging.

## Findings

### [blocking] Fail closed when the changed-file list may be truncated
- where: `pkg/plugins/label/label.go:229-236`
- concern: GitHub's pull-request files endpoint returns at most 3,000 files ([API documentation](https://docs.github.com/en/rest/pulls/pulls#list-pull-requests-files)). This check only verifies the files returned by that endpoint, so on a larger PR a user who can approve every returned file could still receive access without being an approver for an omitted file. Compare against a reliable changed-file count or another complete source, and deny OWNERS access if completeness cannot be confirmed.
- excerpt: |
    changes, err := gc.GetPullRequestChanges(org, repo, e.Number)
    if err != nil {
    	return false, fmt.Errorf("get pull request changes: %w", err)
    }
    userSet := sets.New(user)
    approverForAllFiles = len(changes) > 0
    for _, change := range changes {
    	if approvers.CaseInsensitiveIntersection(owners.Approvers(change.Filename).Set(), userSet).Len() == 0 {

### [nit] Share the OWNERS authorization fallback
- where: `pkg/plugins/label/label.go:298-305`
- concern: The same fallback and denial-message logic appears in the label-removal path at lines 334-341. A shared helper would keep the add and remove authorization paths consistent as they evolve.
- excerpt: |
    if !canSetLabel && e.IsPR && restrictedLabels[labelToAdd].AllowApproversFromOwners {
    	canSetLabel, err = isOwnersApproverForAllFiles()
    	if err != nil {
    		return err
    	}
    	if !canSetLabel {
    		canNotSetLabelReason += " You are also not an OWNERS approver for every file changed by this PR."
    	}
    }

### [nit] Describe the permission as changing labels
- where: `pkg/plugins/config.go:541-542`
- concern: The comment says this option lets approvers “set” the label, but the implementation also lets them remove it. “Change” would describe both operations.
- excerpt: |
    // AllowApproversFromOwners allows a user who can approve every changed PR file to set the label.
    AllowApproversFromOwners bool `json:"allow_approvers_from_owners,omitempty"`

## Checked

- Existing user and team permissions remain in place, and OWNERS access applies only to PRs with the option enabled.
- Empty change lists and OWNERS/GitHub lookup errors do not grant access.
- Both new options default to false, preserving existing configuration behavior.
- Label reset only removes labels explicitly configured with `remove_on_new_commits`.
- Code Quality and Maintainability reviews found no blocking issues; the Deployment Risk review confirmed the 3,000-file limit against GitHub's documentation.

## Open questions

None.
