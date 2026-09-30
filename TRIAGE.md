---
issue: kubernetes-sigs/prow#958
title: "[triggers]: `/retest` does not re-run failed GitHub Actions"
state: closed
labels: ""
main_sha: f56daaa3a55d245be5a03acd0dea5960a62db304
triaged_at: 2026-09-30T12:39:26Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
refresh_log:
  - at: 2026-09-30T12:39:26Z
    since: 2026-09-22T20:27:46Z
    summary: "PR #955 merged and issue #958 closed; updated the saved outcome."
---

## Verdict

Accepted bug, fixed by PR #955. The merged change removes the unsupported multi-event GitHub Actions query that made Prow's opted-in `/retest` workflow discovery return an empty successful result. Issue #958 is closed.

## What the issue reports

- With `trigger_github_workflows: true`, `/retest` does not rerun a failed Actions workflow.
- The reporter reproduced this on released versions and `main`.
- GitHub returns matching runs for `event=pull_request`, but zero for Prow's `pull_request OR pull_request_target OR workflow_call` value.
- Prow logs no error because the query succeeds but returns no runs.

Since previous triage:

- PR #955 merged at 2026-09-30T01:24:49Z as `7626c76a396f367698b87f0650c8c5df14ae6c4e`.
- kubernetes-prow[bot] closed issue #958 at 2026-09-30T01:24:50Z. There were no new issue comments or cross-references.

## Findings

### [reproducibility] The exact Prow event value returns zero runs
- detail: The issue contains live GitHub API output showing a failed run without the filter, four `pull_request` runs with a single event, and zero with Prow's OR-separated value.
- evidence: kubernetes-sigs/prow#958.

### [cause] The Actions event filter accepts one exact event
- detail: `GetFailedActionRunsByHeadBranch` sends one `event` parameter containing three event names joined with `OR`; GitHub treats it as an unknown name and returns no workflow runs.
- evidence: `pkg/github/client.go:2194-2236`.

### [related-code] Retest reruns only the discovered runs
- where: `pkg/plugins/trigger/generic-comment.go:169-196`
- excerpt: |
    failedRuns, err := c.GitHubClient.GetFailedActionRunsByHeadBranch(org, repo, pr.Head.Ref, headSHA)
    if err := c.GitHubClient.TriggerFailedGitHubWorkflow(org, repo, runID); err != nil {
- relevance: An empty list prevents every `rerun-failed-jobs` request.

### [related-code] Pending approval discovery repeats the invalid filter
- where: `pkg/github/client.go:2270-2304`
- excerpt: |
    query.Add("event", "pull_request OR pull_request_target")
    query.Add("status", "action_required")
- relevance: `/ok-to-test` cannot discover pending eligible Actions runs through this request either.

### [related-issue] Issue #957 is a separate approval-path concern
- ref: kubernetes-sigs/prow#957
- relevance: It tracks the broad 403-to-rerun fallback that becomes reachable after pending-run discovery works; do not combine it with the focused #955 fix.

## Resolved

### [related-pr] PR #955 fixes both workflow-run queries
- ref: kubernetes-sigs/prow#955
- relevance: Merged at 2026-09-30T01:24:49Z as `7626c76a396f367698b87f0650c8c5df14ae6c4e`. It removes the invalid API filter, filters events locally, paginates, excludes `action_required` from retries, and adds regression coverage. Issue #958 closed one second later.

## Checked

- The issue is closed, unlabelled, and assigned to #955's author.
- `pkg/plugins/config.go:610-611` defines the opt-in trigger setting.
- `site/content/en/docs/announcements.md:46-53` documents failed-GitHub-job triggering.
- `pkg/github/client_test.go:384-489` previously asserted the invalid event query.
- PR #955 adds event, pagination, and `action_required` cases.

## Next steps

- No further triage action is needed for #958; PR #955 is merged and the issue is closed.
- If production validation is still desired, confirm `/retest` in an opted-in deployment with `actions: write`; an eligible failed run should increment `run_attempt`.
- Track #957's 403 fallback independently.

## Open questions

- Can deployment validation confirm that the production token has sufficient Actions permissions?
