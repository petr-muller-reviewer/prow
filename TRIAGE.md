---
issue: kubernetes-sigs/prow#194
title: "Allow `ok-to-test` label to approve GitHub workflow runs for new contributors #25210"
state: open
labels: kind/feature, help wanted, sig/contributor-experience, area/plugins
main_sha: 4eec6cb88572c8a3d6e82bd9f9cf4e8836c3252f
triaged_at: 2026-09-22T00:20:51Z
verdict: accepted
---

## Findings

### [reproducibility] Actions lookup returns no runs
- detail: @dklima measured that GitHub treats the multi-event `event` value as an exact string; `event=pull_request OR pull_request_target` returns no matches. The same behavior was observed against cluster-api and kubeflow/notebooks.
- evidence: https://github.com/kubernetes-sigs/prow/issues/194#issuecomment-5757126975

### [cause] Invalid multi-event query breaks approval and retest
- detail: `GetPendingApprovalActionRuns()` and `GetFailedActionRunsByHeadBranch()` use unsupported `OR` event filters. The original `/ok-to-test` approval therefore cannot discover runs, and `/retest` cannot discover failed Actions runs.
- evidence: `pkg/github/client.go:2194-2302`

### [cause] Approval only occurs on the comment path
- detail: Approval is invoked for `/ok-to-test` comments, not for later trusted pushes or a human-applied `ok-to-test` label. Those events can create fresh approval-gated runs.
- evidence: `pkg/plugins/trigger/generic-comment.go:147-154`; `pkg/plugins/trigger/pull-request.go:123-151`

### [cause] Approval-gated runs are not failed runs
- detail: A gated run reports `status=completed` and `conclusion=action_required`; correcting lookup must exclude that conclusion from `/retest` candidates.
- evidence: https://github.com/kubernetes-sigs/prow/issues/194#issuecomment-5757126975

### [related-code] Actions workflow-run client
- where: `pkg/github/client.go:2194-2302`
- excerpt: |
    query.Add("event", "pull_request OR pull_request_target")
    query.Add("status", "action_required")

### [related-code] Trigger approval paths
- where: `pkg/plugins/trigger/generic-comment.go:147-154`
- excerpt: |
    if isOkToTest && trigger.TriggerGitHubWorkflows {
        approveGitHubActionsWorkflowRuns(c, org, repo, pr.Head.Ref, headSHA)
    }

### [related-pr] #955 fixes workflow discovery
- ref: kubernetes-sigs/prow#955
- relevance: Removes the unsupported event query, handles pagination, filters supported events client-side, and prevents `/retest` from rerunning `action_required` runs.

### [related-pr] #956 approves runs after qualifying PR events
- ref: kubernetes-sigs/prow#956
- relevance: Depends on #955; adds bounded polling and revalidation for trusted pushes, reopen, ready-for-review, base changes, and human-applied `ok-to-test` labels.

### [related-pr] #612 was incomplete
- ref: kubernetes-sigs/prow#612
- relevance: Added approval for `/ok-to-test` comments but did not handle the invalid lookup or subsequent events.

### [related-issue] Original issue
- ref: kubernetes/test-infra#25210
- relevance: Pre-migration origin of the request.

## Checked

- Issue state, labels, discussion, and the 2026-09-21 root-cause report.
- Current `pkg/github` and trigger-plugin code and its existing unit tests.
- Open PRs #955 and #956; their reported project CI jobs succeeded while Tide remained pending at triage time.
- `trigger_github_workflows` is the existing opt-in gate.

## Next steps

- Review and merge #955 before #956.
- Validate both PRs on a fork PR with `trigger_github_workflows: true`, GitHub App `actions: write`, `/ok-to-test`, and a subsequent push.
- Decide whether #956's bounded in-handler polling and accumulated waiters are acceptable or need queueing/concurrency limits.
- After merge and live validation, close #194 as fixed; consider replacing `kind/feature` with `kind/bug`.

## Open questions

- Is up to roughly 30 seconds of extra hook-handler lifetime acceptable under a burst of qualifying pushes?
- Do all opted-in deployments grant the GitHub App `actions: write`?
- Should a future `workflow_run` webhook design replace polling despite its broader deployment cost?
