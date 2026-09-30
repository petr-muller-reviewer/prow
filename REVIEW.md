---
pr: kubernetes-sigs/prow#955
title: "fix(github): drop invalid event query filter"
head_sha: 40eb181d73bb9f3097db3ae5b052f2a83850c942
base: main
reviewed_at: 2026-09-29T17:56:50Z
verdict: approve
gate:
  decision: hold
  gated_at: 2026-09-29T17:57:32Z
  gated_head_sha: 40eb181d73bb9f3097db3ae5b052f2a83850c942
  reviewed_head_sha: 40eb181d73bb9f3097db3ae5b052f2a83850c942
---

# Review

## Gate

**Hold.** The PR head still matches the saved review, and the substantive GitHub review requests are addressed in the current code. The saved review's operator-documentation finding remains open: this fix activates Actions calls in opted-in deployments, with permission and command-trust behavior that the current configuration documentation does not explain. Add the operator guidance or a suitable release note before merging.

**Gating list:**

- `REVIEW.md` [should-fix], `pkg/plugins/config.go:610-611` and `pkg/plugins/plugin-config-documented.yaml:706-707`: document `/ok-to-test` approval/rerun, trusted-commenter requirements for Actions commands, and Actions read/write permissions. The current descriptions remain the old one-line text.

**Merge risk:** No exported Go interface or configuration schema/default changed. The blast radius is installations with `trigger_github_workflows: true`: previously empty workflow-run queries now yield approvals and retries, potentially increasing GitHub API calls and comment-handler latency. The feature remains opt-in; rollout should monitor 403s, rate limits, and latency.

## Verdict

Approve with suggestions. The invalid GitHub event query is corrected, and no merge-blocking code issue or configuration break was identified. The existing opt-in setting limits exposure, but enabled deployments will begin making Actions calls that previously returned no runs; the command behavior and required permissions should be documented.

## What this PR does

- Lists workflow runs without the unsupported multi-event query, then filters events and conclusions locally across all pages.
- Selects approval for fork PR runs and rerun for same-repository PR runs.
- Limits Actions approval and retry commands to trusted commenters, even when the PR has an `ok-to-test` label.
- Waits for Actions calls to finish before the comment handler returns.
- Adds fake-client support and tests for approval, retry, trust, and waiting behavior.

## Findings

### [should-fix] Explain the enabled Actions behavior to operators
- where: `pkg/plugins/config.go:610-611`
- concern: The current description does not tell operators that `/ok-to-test` approves or reruns pending workflows, while `/retest` retries failed runs only when the commenter is trusted. Update this description and the matching example in `pkg/plugins/plugin-config-documented.yaml:706-707`; include the Actions read/write permission requirement in the operator documentation. The maintainability and deployment reviewers independently raised this gap.
- excerpt: |
    // TriggerGitHubWorkflows enables workflows run by github to be triggered by prow.
    TriggerGitHubWorkflows bool `json:"trigger_github_workflows,omitempty"`

### [nit] Identify the rerun initiator correctly
- where: `pkg/plugins/trigger/generic-comment.go:322-324`
- concern: The comment says a same-repository bot PR is rerun with the bot as the triggering actor. Prow initiates the API rerun; GitHub preserves the original actor's privileges. Clarify this distinction. See [GitHub's rerun documentation](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/re-run-workflows-and-jobs).
- excerpt: |
    // GitHub documents the approve endpoint only for a fork PR. For a PR from the
    // base repository, for example the PR of a bot, a re-run starts the run with
    // the bot as the triggering actor.

## Checked

- `pkg/github/client.go:2196-2303`: event and conclusion filters, retained SHA/branch constraints, pending-approval status filter, and pagination.
- `pkg/plugins/trigger/generic-comment.go:73-213`: commenter trust checks, command routing, and waiting for Actions calls.
- `pkg/plugins/trigger/generic-comment.go:312-368`: same-repository versus fork approval path and error handling.
- `pkg/github/fakegithub/fakegithub.go:1389-1452` and the new trigger tests: failed-run fixtures, call recording, approval cases, and retry cases.
- Focused trigger tests passed with the race detector: `go test ./pkg/plugins/trigger -run 'TestHandleGenericCommentWaitsForActionsCalls|TestApproveWorkflowRunsByRepository|TestTriggerFailedGitHubWorkflows' -race -count=1`.
- `git diff --check` passed for the PR diff. Live GitHub Actions approval and rerun behavior was not tested.
- The configuration field remains an opt-in bool with no schema or default change.
- GitHub's [workflow-run API documentation](https://docs.github.com/en/rest/actions/workflow-runs) confirms Actions permissions and the 1,000-result bound for filtered searches.
- Deployment risk is medium: enabled repositories may see more API calls and longer comment-handler latency. Monitor Actions 403s, rate limits, and handler latency during rollout; consider bounding concurrent run mutations if bursts appear.

## Open questions

None.
