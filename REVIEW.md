---
pr: kubernetes-sigs/prow#955
title: "fix(github): drop invalid event query filter"
head_sha: e7327921f392ccb426739d03e3b353b1105779da
base: main
reviewed_at: 2026-09-30T12:39:35Z
verdict: approve
refresh_log:
  - old_sha: 40eb181d73bb9f3097db3ae5b052f2a83850c942
    new_sha: e7327921f392ccb426739d03e3b353b1105779da
    summary: "Clarified the rerun actor comment, hardened the waiting test, and recorded subsequent approval and merge."
gate:
  decision: hold
  gated_at: 2026-09-29T17:57:32Z
  gated_head_sha: 40eb181d73bb9f3097db3ae5b052f2a83850c942
  reviewed_head_sha: 40eb181d73bb9f3097db3ae5b052f2a83850c942
---

# Review

## Gate

**Historical gate: hold at `40eb181d` on 2026-09-29.** The PR has since merged at `e7327921`. The substantive GitHub review requests were addressed, but the saved review's operator-documentation finding remained open at merge. This gate record describes the earlier recommendation; it was not rerun after the final commits.

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

Since previous review:

- `pkg/plugins/trigger/generic-comment.go:322-325` now correctly identifies Prow as the rerun initiator.
- `pkg/plugins/trigger/generic-comment_test.go:1994-2067` uses `sync.Once` and cleanup to make the waiting test safe if another run is added or the test fails early.
- `cblecker` approved the PR at 2026-09-29T23:02:32Z; the PR was subsequently merged at `e7327921`.

## Findings

### [should-fix] Explain the enabled Actions behavior to operators
- where: `pkg/plugins/config.go:610-611`
- concern: The current description does not tell operators that `/ok-to-test` approves or reruns pending workflows, while `/retest` retries failed runs only when the commenter is trusted. Update this description and the matching example in `pkg/plugins/plugin-config-documented.yaml:706-707`; include the Actions read/write permission requirement in the operator documentation. The maintainability and deployment reviewers independently raised this gap.
- excerpt: |
    // TriggerGitHubWorkflows enables workflows run by github to be triggered by prow.
    TriggerGitHubWorkflows bool `json:"trigger_github_workflows,omitempty"`

## Resolved

### [nit] Identify the rerun initiator correctly
- where: `pkg/plugins/trigger/generic-comment.go:322-325`
- concern: Resolved in `8f60d1cec`. The comment now says the API caller (Prow) initiates the rerun and clears the approval gate.
- excerpt: |
    // base repository, for example the PR of a bot, re-run the workflow instead.
    // The re-run sets the triggering actor to the API caller (prow), which clears
    // the approval gate.

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

## Followups

### Document GitHub Actions triggering behavior

- category: docs
- necessity: should
- where: `pkg/plugins/config.go:610-611`, `pkg/plugins/plugin-config-documented.yaml:706-707`, `site/content/en/docs/jobs.md:265-282`, `site/content/en/docs/announcements.md:48`

```text
In kubernetes-sigs/prow, following merged PR #955, "fix(github): drop invalid event query filter" (merge commit 7626c76a396f367698b87f0650c8c5df14ae6c4e), document the behavior that `trigger_github_workflows` now actually enables on the merged default branch.

Background: before #955 both workflow-run queries always returned zero runs, so the flag had no visible effect. After #955, installations with the flag enabled approve or rerun GitHub Actions:
- A trusted commenter's `/ok-to-test` approves the pending runs of the PR head SHA when the PR comes from a fork, and reruns them when the PR head is in the base repository.
- A trusted commenter's `/retest` or `/test all` reruns completed runs whose conclusion is `failure`, `cancelled`, `timed_out`, or `startup_failure`, for `pull_request` and `pull_request_target` runs only.
- The `ok-to-test` label staying on a PR does not let an untrusted author start Actions runs; ProwJobs may still start.

Update the `TriggerGitHubWorkflows` field comment in pkg/plugins/config.go, its matching example in pkg/plugins/plugin-config-documented.yaml, and the comment-triggered-jobs section in site/content/en/docs/jobs.md. State that the flag is opt-in, which commands affect Actions and who may issue them, and that Prow needs GitHub Actions read permission to list runs and write permission to approve or rerun them; link GitHub's workflow-run REST API documentation. Add a dated entry under "New features" in site/content/en/docs/announcements.md describing the production-visible change for installations that already set the flag; the existing 2022 entry ("Trigger plugin can now trigger failed github jobs", line 48) is now incomplete.

Check the status of PR #956 ("feat(trigger): approve workflow runs on push") before writing. As of 2026-11-16 it is still OPEN, so document only the post-#955 behavior and do not promise automatic approval after a push.

Acceptance criteria: the two config descriptions, the jobs guide, and the announcement agree with the merged implementation; an operator can tell which commands affect Actions, who can issue them, and which permissions are required. Documentation only: do not change trigger logic, API calls, or tests, and do not post to GitHub unless separately asked.
```

### Fix misleading logging in the Actions path

- category: cleanup
- necessity: could
- where: `pkg/github/client.go:2353`, `pkg/plugins/trigger/generic-comment.go:177`

```text
In kubernetes-sigs/prow, following merged PR #955, "fix(github): drop invalid event query filter" (merge commit 7626c76a396f367698b87f0650c8c5df14ae6c4e), fix two logging defects in the GitHub Actions path.

- In pkg/github/client.go, `GetFailedActionRunsByHeadBranch` opens its duration logger with the span name "GetJobsByHeadBranch", a name that no longer matches the function. Rename it to "GetFailedActionRunsByHeadBranch" so logs and timing metrics identify the call correctly.
- In pkg/plugins/trigger/generic-comment.go, the "headSHA unavailable" branch logs the entire `pr` struct with `%v` instead of the error, which hides the actual cause. Log `err`, and include org, repo, and the PR number or pr.Head.Ref; make the message say that GitHub Actions runs for the PR will not be rerun.

Acceptance criteria: `go build ./pkg/github/... ./pkg/plugins/trigger/...` succeeds and `go test ./pkg/github/... ./pkg/plugins/trigger/... -count=1` passes. The emitted messages name the failing call and include the underlying error. No behavior change.
Scope guard: do not restructure the call flow, change trust logic, or introduce a new logging framework.
```

### Centralize PR workflow-run listing

- category: tech-debt
- necessity: could
- where: `pkg/github/client.go:2352-2470` (`GetFailedActionRunsByHeadBranch`, `listWorkflowRuns`, `GetPendingApprovalActionRuns`)

```text
In kubernetes-sigs/prow, following merged PR #955, "fix(github): drop invalid event query filter" (merge commit 7626c76a396f367698b87f0650c8c5df14ae6c4e), extract the duplicated PR workflow-run lookup from `GetFailedActionRunsByHeadBranch` and `GetPendingApprovalActionRuns`.

Both getters build the same `url.Values` (`per_page=100`, `head_sha`, `branch`), call `listWorkflowRuns`, and then keep only runs whose Event is `pull_request` or `pull_request_target` (`pullRequestWorkflowRunEvents`). The constraint that GitHub cannot filter multiple events server-side is documented only on `listWorkflowRuns`, so each caller must remember the client-side filter.

Add an unexported helper, for example `listPullRequestWorkflowRuns(org, repo, branchName, headSHA string, extra url.Values) ([]WorkflowRun, error)`, that merges the extra parameters with the shared ones, pages through `listWorkflowRuns`, and returns only PR-event runs. Use it in both getters and keep their own predicates: completed plus a retestable conclusion in `GetFailedActionRunsByHeadBranch`, the `status=action_required` query parameter in `GetPendingApprovalActionRuns`. Keep a doc comment explaining why event filtering is client-side, so a future caller cannot reintroduce the bug #955 fixed.

Acceptance criteria: `go test ./pkg/github/... -count=1` passes without changing test expectations; `TestGetFailedActionRunsByHeadBranch`, `TestGetFailedActionRunsByHeadBranchPagination`, `TestGetPendingApprovalActionRuns`, and `TestGetPendingApprovalActionRunsPagination` still exercise event filtering and pagination.
Scope guard: no behavior change, no new query parameters, no changes to the trigger plugin.
```
