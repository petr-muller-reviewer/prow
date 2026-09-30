---
issue: kubernetes-sigs/prow#957
title: "The 403 fallback of the workflow approval accepts every 403"
state: closed
labels: []
main_sha: 7626c76a396f367698b87f0650c8c5df14ae6c4e
triaged_at: 2026-09-30T13:33:08Z
verdict: accepted
refresh_log:
  - at: 2026-09-30T12:38:56Z
    since: 2026-09-21T08:57:28Z
    summary: "PR #955 merged the topology-based fix; issue #957 closed as completed."
---

# Triage

## Verdict

**Accepted bug, appropriately fixed in code and closed.** PR #955 merged at 2026-09-30T01:24:49Z with the topology-based correction and `Fixes #957`; GitHub closed the issue as completed at 2026-09-30T01:24:51Z. The merged code chooses re-run for a same-repository PR and approval for a fork PR before attempting either endpoint. Focused tests pass; a live blocked-run test was not reported.

## What the issue reports

- The original trigger fallback reran pending workflows after any residual approval 403.
- Other authorization failures could also produce that 403, so the fallback was broader than its intended same-repository bot case.
- The invalid pending-run query meant this path had been dormant until #955 fixed the query.

Since previous triage:

- PR #955 merged the query and fallback fixes; issue #957 closed automatically. No new issue comments or labels appeared.

## Findings

### [related-code] Merged code chooses the operation from PR topology
- where: `pkg/plugins/trigger/generic-comment.go:312-361`
- excerpt: |
    return pr.Head.Repo.FullName != "" && strings.EqualFold(pr.Head.Repo.FullName, pr.Base.Repo.FullName)
- relevance: Fork PRs take the approval path; approval errors, including 403, are logged without a re-run.

### [related-code] Workflow calls require a trusted commenter
- where: `pkg/plugins/trigger/generic-comment.go:148-155`
- excerpt: |
    if isOkToTest && trigger.TriggerGitHubWorkflows {
        if trustedResponse.IsTrusted {
- relevance: A PR author cannot use an existing `ok-to-test` label to approve new workflow runs after a push.

### [reproducibility] Focused unit tests pass
- detail: `go test ./pkg/plugins/trigger -run 'Test(IsSameRepoPullRequest|ApproveWorkflowRunsByRepository)$' -count=1` passed. Cases cover same-repository rerun, fork approval 403 without rerun, case-insensitive names, and absent head-repository information.
- evidence: `pkg/plugins/trigger/generic-comment_test.go:2081-2250`.

### [related-pr] #955 merged and closed the issue
- ref: kubernetes-sigs/prow#955
- relevance: Merged at 2026-09-30T01:24:49Z as `7626c76a396f367698b87f0650c8c5df14ae6c4e`; includes fix commit `8f60d1cec2b2` and `Fixes #957`.

### [related-pr] #956 remains open
- ref: kubernetes-sigs/prow#956
- relevance: Extends approval to push events and depends on the behavior merged in #955.

## Resolved

### [reproducibility] Every residual 403 invoked rerun before #955
- detail: On main SHA `cd1c1dbd246180e7183af5eae6cc68c2053ffcdc`, the helper used `github.IsForbidden(err)` to rerun. #955 removed this fallback and selects the endpoint from PR topology before the call.
- evidence: Historical `pkg/plugins/trigger/generic-comment.go:336-344`; merged fix `pkg/plugins/trigger/generic-comment.go:312-361`.

### [cause] Forbidden errors did not encode the reason for refusal
- detail: The client still classifies residual 403 responses broadly, but the approval helper no longer uses that classification to decide whether to rerun.
- evidence: Historical `pkg/github/client.go:923-941`; merged fix commit `8f60d1cec2b2`.

### [cause] Rerun is not equivalent to approval
- detail: The original fallback conflated these operations; #955 makes their use depend on PR topology.
- evidence: `pkg/plugins/trigger/generic-comment.go:312-361`.

### [related-pr] #798 introduced the fallback
- ref: kubernetes-sigs/prow#798
- relevance: Added the former rerun-after-403 behavior for bot-created same-repository PRs; #955 replaced it.

## Checked
- Confirmed the reported fallback and error classification on main SHA `cd1c1dbd246180e7183af5eae6cc68c2053ffcdc`.
- Confirmed #798 tests explicitly expect a rerun after generic `NewForbidden()`.
- Confirmed #955's invalid `event=pull_request OR pull_request_target` query is present in `pkg/github/client.go:2274-2301`.
- Confirmed PR #955's merged commit and closing reference; current issue state is closed as completed, with no labels or new comments.
- Confirmed the merged helper branches on `isSameRepoPullRequest` in `pkg/plugins/trigger/generic-comment.go:312-361`.
- Ran the focused Go tests above successfully against the merged code.
- Checked [GitHub's approve endpoint documentation](https://docs.github.com/en/rest/actions/workflow-runs#approve-a-workflow-run-for-a-fork-pull-request): it is scoped to public-fork PRs. [GitHub's rerun documentation](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/re-run-workflows-and-jobs) says reruns retain the original actor's privileges; the original concern is bypassing approval, not gaining the rerunner's token permissions.

## Next steps
- No further action on #957; it is fixed and closed.
- If operational validation is desired, test an `action_required` same-repository run in a configured deployment during rollout of #955/#956.

## Open questions
- None blocking closure of #957. PR #955 reports unit tests only; deployment behavior has not been confirmed in this triage.
