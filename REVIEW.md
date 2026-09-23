---
pr: kubernetes-sigs/prow#956
title: "feat(trigger): approve workflow runs on push"
head_sha: d40b843e050f907b9f06a3c6720603c25c6fc94d
base: main
reviewed_at: 2026-09-22T20:42:53Z
verdict: request-changes
refresh_log:
  - old_sha: bf3512b7239122f080a629a4ee5943e47df464fd
    new_sha: d40b843e050f907b9f06a3c6720603c25c6fc94d
    summary: "Incorporated boilerplate-header corrections in the workflow approval source and test."
---

## What this PR does

- Approves pending GitHub Actions runs for trusted PR lifecycle events.
- Polls Actions after a webhook, revalidating the head SHA and trust before approval.
- Keeps approval work within hook shutdown tracking.

Since previous review:

- Corrected only the boilerplate copyright headers in `pkg/plugins/trigger/workflow-approval.go` and `pkg/plugins/trigger/workflow-approval_test.go` (2 insertions, 2 deletions); the polling finding is unaffected.

## Findings

### [should-fix] Keep polling after approving the first batch
- where: `pkg/plugins/trigger/workflow-approval.go:122-140`
- concern: The next list response can still contain the just-approved run with `action_required`. Because its ID is already in `attempted`, `pending` is empty and the function returns at line 129. A workflow run created after that second request is never observed or approved. This also happens if an already-started `pull_request_target` run makes the first non-empty response arrive before a blocked `pull_request` run. The stated five-attempt window therefore ends as soon as any non-action-required response is seen, rather than covering delayed sibling runs. Continue polling through the intended settlement window (while retaining the per-run `attempted` guard), and add a case where the second response only has the already-attempted run but a new pending sibling arrives later.
- excerpt: `if github.IsPendingApprovalRun(run) && !attempted.Has(run.ID) { pending = append(pending, run) }; if len(pending) == 0 { return true, nil }`

## Checked
- Event selection covers synchronize, reopen, ready-for-review, base edits, and human-applied `ok-to-test`; it excludes opened and LGTM events.
- Trust and head SHA are revalidated before each approval batch.
- The approval work is deferred until ProwJobs are handled and still runs for repositories without presubmits.
- The API query is scoped by head SHA and only pull-request workflow events are eligible.
- `go test ./pkg/plugins/trigger/... ./pkg/github/... -count=1` passed.

## Open questions
- None.
