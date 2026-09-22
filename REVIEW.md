---
pr: kubernetes-sigs/prow#955
title: "fix(github): drop invalid event query filter"
head_sha: 60ea22ac64543b06ffa6b8ffb6cfc3bfce6b4e4e
base: main
reviewed_at: 2026-09-21T22:35:14Z
verdict: approve
---

## Verdict

Approve. The unsupported multi-event API query is replaced with explicit event filtering after retrieval, and the new pagination and `action_required` handling preserve the intended approval and retry behavior.

## What this PR does

- Retrieves workflow runs without GitHub's invalid multi-event `event` query.
- Filters retryable and approval-eligible events in the client.
- Reads all workflow-run result pages.
- Keeps `action_required` runs out of `/retest` candidates.

## Findings

### Blocking

None.

### Should fix

None.

### Nits

None.

### Questions

None.

## Checked

- `pkg/github/client.go:2209-2260`: both call paths retain `head_sha` and branch constraints, use the shared paginated reader, and filter only the required events client-side.
- `pkg/github/client.go:2226-2231`: completed `action_required` runs are excluded from failed-run retries.
- `pkg/github/client.go:2302-2318`: pending-approval lookup retains the API's `status=action_required` filter, then admits only pull-request events.
- `pkg/github/client_test.go:523-614`: pagination tests verify filters persist on the next-link request and that eligible runs on later pages are returned.
- `go test ./pkg/github -run 'TestGet(FailedActionRunsByHeadBranch|PendingApprovalActionRuns)' -count=1` passed.
- `git diff --check upstream/main...HEAD` passed.
- Independent code-quality, maintainability, and deployment-risk reviews all recommended approval; the only suggestion was a future predicate/lookup-set refactor if more event policies are added.

## Open questions

None.
