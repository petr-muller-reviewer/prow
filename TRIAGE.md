---
issue: kubernetes-sigs/prow#194
title: "Allow `ok-to-test` label to approve GitHub workflow runs for new contributors #25210"
state: open
labels: kind/feature, help wanted, sig/contributor-experience, area/plugins
main_sha: 4eec6cb88572c8a3d6e82bd9f9cf4e8836c3252f
triaged_at: 2026-11-03T22:13:21Z
verdict: accepted
refresh_log:
  - at: 2026-11-03T22:13:21Z
    since: 2026-09-22T00:20:51Z
    summary: "#955 merged 2026-09-30; #956 rebased onto post-#955 main and still open with green CI; no new issue comments, labels, or cross-references."
advice:
  advised_at: 2026-11-03T22:14:08Z
  based_on_triaged_at: 2026-11-03T22:13:21Z
---

## What the issue reports

- GitHub Actions workflow runs for new contributors stay at `action_required` until a maintainer approves them, even though Prow's `ok-to-test` label already grants trust.
- `/ok-to-test` should approve the pending runs, including for later trusted pushes and a human-applied `ok-to-test` label, under the existing `trigger_github_workflows` opt-in.
- Migrated from kubernetes/test-infra#25210.

Since previous triage:

- #955 merged on 2026-09-30, fixing the workflow-run lookup and `/retest` candidate filtering.
- #956 remains open, rebased onto post-#955 `main`, mergeable, with all required CI green and Tide still pending; no reviews yet.
- No new issue comments, labels, or cross-references since 2026-09-22.

## Findings

### [reproducibility] Actions lookup returns no runs
- detail: @dklima measured that GitHub treats the multi-event `event` value as an exact string; `event=pull_request OR pull_request_target` returns no matches. The same behavior was observed against cluster-api and kubeflow/notebooks.
- evidence: https://github.com/kubernetes-sigs/prow/issues/194#issuecomment-5757126975

### [cause] Approval only occurs on the comment path
- detail: Approval is invoked for `/ok-to-test` comments, not for later trusted pushes or a human-applied `ok-to-test` label. Those events can create fresh approval-gated runs.
- evidence: `pkg/plugins/trigger/generic-comment.go:147-154`; `pkg/plugins/trigger/pull-request.go:123-151`
- status: Unchanged; addressed by open #956, not yet merged.

### [related-code] Trigger approval paths
- where: `pkg/plugins/trigger/generic-comment.go:147-154`
- excerpt: |
    if isOkToTest && trigger.TriggerGitHubWorkflows {
        approveGitHubActionsWorkflowRuns(c, org, repo, pr.Head.Ref, headSHA)
    }

### [related-pr] #955 fixes workflow discovery
- ref: kubernetes-sigs/prow#955
- relevance: Merged 2026-09-30 (commit b934e31c7, merge 7626c76a3). Removes the unsupported event query, handles pagination, filters supported events client-side, and prevents `/retest` from rerunning `action_required` runs.

### [related-pr] #956 approves runs after qualifying PR events
- ref: kubernetes-sigs/prow#956
- relevance: Rebased onto post-#955 `main` (base ae4792128, head 47b3b7911); adds bounded polling and revalidation for trusted pushes, reopen, ready-for-review, base changes, and human-applied `ok-to-test` labels. Open, mergeable, all required CI green, Tide pending, no reviews.

### [related-pr] #612 was incomplete
- ref: kubernetes-sigs/prow#612
- relevance: Added approval for `/ok-to-test` comments but did not handle the invalid lookup or subsequent events.

### [related-issue] Original issue
- ref: kubernetes/test-infra#25210
- relevance: Pre-migration origin of the request.

## Resolved

### [cause] Invalid multi-event query breaks approval and retest
- detail: `GetPendingApprovalActionRuns()` and `GetFailedActionRunsByHeadBranch()` used an unsupported `OR` event filter, so both returned no runs.
- resolved: #955 merged 2026-09-30 (commit b934e31c7, merge 7626c76a3); the `event` parameter is dropped, results are paginated and filtered client-side, and `action_required` is excluded from `/retest` candidates.

### [related-code] Actions workflow-run client
- where: `pkg/github/client.go` (`listWorkflowRuns`, `GetFailedActionRunsByHeadBranch`)
- excerpt: |
    runs, err := c.listWorkflowRuns(org, repo, url.Values{
        "per_page": []string{"100"},
        "head_sha": []string{headSHA},
        "branch":   []string{branchName},
    })
- resolved: Replaced by #955; the original invalid-query excerpt no longer exists on `main`.

### [cause] Approval-gated runs are not failed runs
- detail: A gated run reports `status=completed` and `conclusion=action_required`; the corrected lookup had to exclude that conclusion from `/retest` candidates.
- resolved: #955 adds `action_required` to `nonRetestableWorkflowRunConclusions`, so approval-gated runs are not re-run by `/retest`.

## Checked

- Issue state, labels, discussion, and the 2026-09-21 root-cause report.
- Current `pkg/github` and trigger-plugin code and its existing unit tests.
- Open PRs #955 and #956; their reported project CI jobs succeeded while Tide remained pending at triage time.
- `trigger_github_workflows` is the existing opt-in gate.
- Refresh on 2026-11-03: issue state/labels/comments/timeline, #955 merge, #956 rebase/CI/reviews, and upstream/main movement (173 commits since 4eec6cb; only #955 relevant).

## Next steps

- Review and merge #956 (#955 is merged; #956 is rebased, mergeable, and green apart from Tide).
- Validate both PRs on a fork PR with `trigger_github_workflows: true`, GitHub App `actions: write`, `/ok-to-test`, and a subsequent push.
- Decide whether #956's bounded in-handler polling and accumulated waiters are acceptable or need queueing/concurrency limits.
- After merge and live validation, close #194 as fixed; consider replacing `kind/feature` with `kind/bug`.

## Advice

Current state: #194 is open, unchanged labels, assigned to @AaruniAggarwal since 2025-10-22 (by k8s-ci-robot). #955 merged 2026-09-30. #956 is open, mergeable, two commits, no reviews, all required CI green, Tide pending.

1. Review and approve #956. It is the only remaining gap: the `[cause] Approval only occurs on the comment path` finding is unchanged, and #956 adds approval after trusted pushes, reopen, ready-for-review, base changes, and a human-applied `ok-to-test` label.

```
gh pr checkout 956
go test ./pkg/plugins/trigger/... ./pkg/github/...
gh pr review 956 --repo kubernetes-sigs/prow --approve --body "LGTM

/approve
/lgtm"
```

2. Post a status comment on #194 and confirm the assignee is still active. No issue comment has arrived since 2026-09-21, and the stale assignment predates both PRs.

```
gh issue comment 194 --repo kubernetes-sigs/prow --body "> *This was generated by AI during triage.*

#955 (workflow-run lookup fix) merged on 2026-09-30. #956 (approval on trusted pushes and the ok-to-test label) is rebased on post-#955 main, mergeable, and green apart from Tide; it is waiting for review. @dklima, thanks for the fix. @AaruniAggarwal, are you still working on this?"
```

3. After #956 merges, validate on a fork PR with `trigger_github_workflows: true`, GitHub App `actions: write`, `/ok-to-test`, and a subsequent push. Check the run conclusions directly:

```
gh api 'repos/<fork-owner>/<fork-repo>/actions/runs?head_sha=<sha>&per_page=100' --jq '.workflow_runs[] | {id,event,status,conclusion}'
```

4. Only after live validation: relabel as a bug and close #194 as fixed.

```
gh issue edit 194 --repo kubernetes-sigs/prow --remove-label kind/feature --add-label kind/bug
gh issue close 194 --repo kubernetes-sigs/prow --comment "> *This was generated by AI during triage.*

Fixed by #955 and #956. Verified on a fork PR with trigger_github_workflows enabled: /ok-to-test, later pushes, and a human-applied ok-to-test label all approve the pending workflow runs."
```

No label additions are needed now: `help wanted`, `sig/contributor-experience`, and `area/plugins` are already present. No milestone or duplicate candidate applies.

## Open questions

- Is up to roughly 30 seconds of extra hook-handler lifetime acceptable under a burst of qualifying pushes?
- Do all opted-in deployments grant the GitHub App `actions: write`?
- Should a future `workflow_run` webhook design replace polling despite its broader deployment cost?
