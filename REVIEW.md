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
- where: `pkg/plugins/config.go:610-611`, `pkg/plugins/plugin-config-documented.yaml:706-707`, `site/content/en/docs/jobs.md:240-246`

```text
In kubernetes-sigs/prow, following merged PR #955, "fix(github): drop invalid event query filter" (merge commit 7626c76a396f367698b87f0650c8c5df14ae6c4e), document the behavior enabled by trigger_github_workflows on the merged default branch.

Update the TriggerGitHubWorkflows field comment in pkg/plugins/config.go, its matching example in pkg/plugins/plugin-config-documented.yaml, and the user-facing command guidance in site/content/en/docs/jobs.md. Explain that the flag is opt-in; a trusted /ok-to-test commenter starts pending Actions runs (approve for fork PRs, rerun for same-repository PRs); and a trusted /retest or /test all commenter reruns eligible failed Actions runs. Explain that an ok-to-test label on a PR does not grant its untrusted author permission to start Actions runs, even though ProwJobs can still start. State the GitHub Actions read permission needed to list runs and write permission needed to approve or rerun them, linking to GitHub's workflow-run REST API documentation.

Check the status of PR #956 before writing. If it has merged, include its push-approval behavior as implemented; if it has not, document only the behavior present after #955 and avoid promising automatic approval after a push.

Acceptance criteria: the two config descriptions and jobs guide agree with the merged implementation; operators can tell which commands affect Actions, who can issue them, and which permissions are required. Keep this followup documentation-only: do not change trigger logic, API calls, or tests. Do not post to GitHub unless separately asked.
```
