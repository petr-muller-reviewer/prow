---
pr: kubernetes-sigs/prow#956
title: "feat(trigger): approve workflow runs on push"
head_sha: ecb089550f328ef502f7c752288a60e04389fbe5
base: main
reviewed_at: 2026-09-30T15:15:16Z
verdict: request-changes
---

# Review

## Verdict

Request changes. All three maintainer perspectives independently found that polling can stop after the first approval batch and miss a workflow run that appears slightly later. The feature remains opt-in and needs no config migration, but enabled installations should account for additional Actions API traffic and longer-lived trigger handlers.

## What this PR does

- Starts GitHub Actions approval work for eligible, trusted pull-request events after the webhook is handled.
- Polls for workflow runs matching the pull request's branch and head SHA.
- Approves pending fork runs and reruns eligible same-repository runs.
- Rechecks the pull request's head SHA and trust before acting, and tracks the work during hook shutdown.

## Findings

### [blocking] Keep polling after the first approval batch

- where: `pkg/plugins/trigger/workflow-approval.go:127-128`
- concern: After approving the first visible run, the next list response can still contain only that run. Its ID is already in `attempted`, so `pending` is empty and the loop exits before a sibling run appears on a later poll, leaving that run at `action_required`. Continue observing for a bounded settlement period after an approval, and add a regression test where the sibling first appears on the third list response.
- excerpt: |
    if len(pending) == 0 {
        return true, nil
    }

### [should-fix] Document the cost of enabling workflow approval

- where: `pkg/plugins/trigger/workflow-approval.go:33-37`
- concern: An eligible event can hold a trigger handler for about 30 seconds and make up to five paginated Actions-list requests, even when no workflow run is created. Operator-facing guidance should describe the added API load and the Actions permission needed for approval or rerun, so installations can plan before enabling the existing flag.
- excerpt: |
    // These values give 5 attempts over approximately 30 seconds.
    const (
        workflowRunPollInterval = 2000
        workflowRunPollSteps    = 5
    )

### [nit] Make poll-exhaustion logging more diagnostic

- where: `pkg/plugins/trigger/workflow-approval.go:141-142`
- concern: The same log message covers a PR with no workflow runs and runs that stayed at the approval gate. Distinguishing those outcomes, ideally with attempt and approval counts, would make incidents easier to diagnose.
- excerpt: |
    if wait.Interrupted(err) {
        log.Info("Gave up waiting for the workflow runs to leave the approval gate.")
    }

## Checked

- Reviewed the two feature commits in the merge diff against `upstream/main` (merge base `e7327921f392ccb426739d03e3b353b1105779da`).
- Inspected event selection, trust and head-SHA revalidation, Actions run filtering, and deferred approval handling.
- Existing polling test introduces a delayed sibling on the second list response, but does not cover an already-attempted run alone on that response.
- The `trigger_github_workflows` setting remains a default-off bool; no configuration migration was identified. Approval work is deferred until after ProwJob handling, and hook shutdown waits for it.
- PR CI checks, including unit tests, race detector, lint, and integration tests, were passing on September 30. Local tests were not run at this head.

## Open questions

- None.
