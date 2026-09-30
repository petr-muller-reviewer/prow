---
issue: kubernetes-sigs/prow#957
title: "The 403 fallback of the workflow approval accepts every 403"
state: open
labels: []
main_sha: cd1c1dbd246180e7183af5eae6cc68c2053ffcdc
triaged_at: 2026-09-21T08:57:28Z
verdict: accepted
---

## Findings

### [reproducibility] Every residual 403 invokes rerun
- detail: The approval helper falls back after `github.IsForbidden(err)`. The existing test uses `github.NewForbidden()` and expects the workflow to be rerun, so the broad condition is deterministic and covered as current behaviour.
- evidence: `pkg/plugins/trigger/generic-comment.go:336-344`; `pkg/plugins/trigger/generic-comment_test.go:2005-2110`.

### [cause] Forbidden errors do not encode the reason for refusal
- detail: `IsForbidden` tests only whether the error is `forbiddenError`. Request handling retries rate/abuse 403s and turns a scope mismatch into a normal error, but all other 403 responses become this type. Thus permission, SSO/IP policy, or installation-state failures are indistinguishable from the endpoint's non-fork refusal at the plugin call site.
- evidence: `pkg/github/client.go:923-941`; `pkg/github/client.go:1049-1112`.

### [cause] Rerun is not equivalent to approval
- detail: The approve and rerun endpoints are distinct operations. The fallback changes the triggering actor, so a generic authorization failure must not be treated as evidence that rerun is safe.
- evidence: `pkg/github/client.go:2241-2254`; `pkg/github/client.go:2304-2320`.

### [related-code] The caller already has PR topology
- where: `pkg/plugins/trigger/generic-comment.go:137-154`
- excerpt: |
    pr, err := refGetter.PullRequest()
    ...
    approveGitHubActionsWorkflowRuns(c, org, repo, pr.Head.Ref, headSHA)
- relevance: The caller can pass same-repository/fork information from `pr.Head.Repo` to constrain the fallback without parsing an opaque error body.

### [related-pr] #798 introduced the fallback
- ref: kubernetes-sigs/prow#798
- relevance: Added the rerun after every `IsForbidden` result for bot-created same-repository PRs.

### [related-pr] #955 makes the path reachable
- ref: kubernetes-sigs/prow#955
- relevance: Removes the invalid multi-event query from `GetPendingApprovalActionRuns`; currently the query returns no matching runs in practice. Fix this issue before or with #955.

### [related-pr] #956 expands approval attempts
- ref: kubernetes-sigs/prow#956
- relevance: Depends on #955 and must use the corrected fallback decision rather than duplicate the current predicate.

## Checked
- Confirmed the reported fallback and error classification on main SHA `cd1c1dbd246180e7183af5eae6cc68c2053ffcdc`.
- Confirmed #798 tests explicitly expect a rerun after generic `NewForbidden()`.
- Confirmed #955's invalid `event=pull_request OR pull_request_target` query is present in `pkg/github/client.go:2274-2301`.
- Searched related Prow issues; #957 is the specific record for this condition.
- Verified available recommended labels: `kind/bug`, `area/plugins`, `help wanted`.

## Next steps
- Accept as a bug and label `kind/bug`, `area/plugins`, and `help wanted` if desired.
- Prefer a topology-gated fallback: rerun only after confirming the PR is same-repository; log and retain a forbidden error for forks or unknown topology.
- Alternatively, add a narrow client-level predicate for a stable documented non-fork API response; do not expose generic body matching at the plugin layer without response-level tests.
- Add tests for same-repository fallback and for fork/unauthorized-policy 403s that must not rerun; ensure #956 shares the decision point.
- Validate in a deployment using `trigger_github_workflows: true` and `actions: write`.

## Open questions
- Is GitHub's non-fork response body a stable documented discriminator, or should Prow use PR topology exclusively?
- Is same-repository topology necessary and sufficient for rerun, or should it also require a typed non-fork error?
- Does an integration test confirm that rerunning a fork PR cannot elevate token or secrets access relative to approval?
