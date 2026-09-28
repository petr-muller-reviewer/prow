---
pr: kubernetes-sigs/prow#653
title: 'Add "ok-to-test cancel" command'
head_sha: d853da769b994cf3073572873a601947126fa5db
base: main
reviewed_at: 2026-09-28T17:01:58Z
verdict: request-changes
---

# Review

## Verdict

Request changes. Cancellation can leave approval active, lose the review-needed label, or put that label on a PR that remains trusted through its author. The new trust transition also has no regression tests. PR #653 is already merged; these are follow-up fixes. No configuration migration or new permission is needed, but cancellation does not stop jobs or approved workflows already running.

## What this PR does

- Recognizes `/ok-to-test cancel` in trigger comments.
- Restricts revocation to trusted users.
- Removes `ok-to-test` and adds `needs-ok-to-test` when canceling.
- Documents the command in trigger help.

## Findings

### [blocking] A combined approval and cancellation leaves the PR approved

- where: `pkg/plugins/trigger/generic-comment.go:106-129`
- concern: A trusted comment containing both `/ok-to-test` and `/ok-to-test cancel` first adds `ok-to-test`. The cancel branch checks the label snapshot fetched before that addition, so it does not remove the newly added label. The handler returns while the PR remains trusted and later commits can trigger CI, defeating the explicit cancellation. Handle the cancel directive before approval or make the commands mutually exclusive and settle on one final state; add a regression case.
- excerpt: |
    if isOkToTest && !github.HasLabel(labels.OkToTest, l) {
        if err := c.GitHubClient.AddLabel(org, repo, number, labels.OkToTest); err != nil {
            return err
        }
    }
    // ...
    if isOkToTestCancel {
        // ...
        if github.HasLabel(labels.OkToTest, l) {
            if err := c.GitHubClient.RemoveLabel(org, repo, number, labels.OkToTest); err != nil {
                return err
            }
        }

### [blocking] Cancellation can remove both trust labels

- where: `pkg/plugins/trigger/generic-comment.go:112-130`
- concern: If a PR already has both `ok-to-test` and `needs-ok-to-test` (a state covered by `pull-request_test.go`), a trusted `/ok-to-test cancel` removes `needs-ok-to-test` in the ordinary label cleanup, then removes `ok-to-test`. The final check still sees `needs-ok-to-test` in the old label snapshot and skips restoring it. The PR becomes untrusted, but no longer displays the promised review-needed label. Handle cancellation before ordinary label cleanup or track the updated label state; test this existing state.
- excerpt: |
    if (isOkToTest || github.HasLabel(labels.OkToTest, l)) && github.HasLabel(labels.NeedsOkToTest, l) {
        if err := c.GitHubClient.RemoveLabel(org, repo, number, labels.NeedsOkToTest); err != nil {
            return err
        }
    }
    // ...
    if !github.HasLabel(labels.NeedsOkToTest, l) {
        if err := c.GitHubClient.AddLabel(org, repo, number, labels.NeedsOkToTest); err != nil {
            return err
        }
    }

### [blocking] Cancellation can mark an inherently trusted PR as needing approval

- where: `pkg/plugins/trigger/generic-comment.go:124-132`
- concern: The cancel branch adds `needs-ok-to-test` even when the PR never had an `ok-to-test` grant to revoke. If the author is a trusted org member or collaborator, `TrustedPullRequest` continues to trust the author and run automatic jobs on updates, while the new label can exclude the PR from Tide's documented merge query. Only add the review-needed label when the PR actually loses label-based trust; cover a trusted-author PR in tests.
- excerpt: |
    if github.HasLabel(labels.OkToTest, l) {
        if err := c.GitHubClient.RemoveLabel(org, repo, number, labels.OkToTest); err != nil {
            return err
        }
    }
    if !github.HasLabel(labels.NeedsOkToTest, l) {
        if err := c.GitHubClient.AddLabel(org, repo, number, labels.NeedsOkToTest); err != nil {
            return err
        }
    }

### [blocking] The cancellation transition has no regression tests

- where: `pkg/plugins/trigger/generic-comment.go:118-135`
- concern: The PR adds no cases to the existing table-driven comment-handler tests for this new trust-changing branch. The two label-state failures above would have been exposed by cases for mixed commands and both labels. Add cases for trusted and untrusted commenters, approval present or absent, trusted authors, `IgnoreOkToTest`, and repeated delivery.
- excerpt: |
    isOkToTestCancel := HonorOkToTest(trigger) && pjutil.OkToTestCancelRe.MatchString(textToCheck)
    if isOkToTestCancel {
        if !trustedResponse.IsTrusted {
            resp := "Only trusted users can revoke `/ok-to-test`."
            return c.GitHubClient.CreateComment(org, repo, number, plugins.FormatResponseRaw(gc.Body, gc.HTMLURL, gc.User.Login, resp))
        }

## Checked

- Compared the PR commit against its parent and checked issue #652 and the PR description.
- Reviewed the label-based trust check on subsequent PR updates and the existing comment-handler tests.
- `go test ./pkg/plugins/trigger ./pkg/pjutil` passes.
- Found no applicable documented Go coding-standard violation or persuasive code smell in the diff.
- No configuration, default, permission, or webhook subscription migration is required.
- Two reviewer perspectives independently found the mixed-command failure; two independently found the trusted-author label mismatch.

## Open questions

- Is cancellation intended to affect only future trust checks? The help text should say that it does not stop jobs already running or undo approved GitHub Actions workflows.

## Followups

No followups accepted (4 skipped).
