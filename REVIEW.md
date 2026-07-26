---
pr: kubernetes-sigs/prow#798
title: "fix(trigger): fall back to rerun when approve fails for non-fork PRs"
head_sha: dd76a90690dded89117e44ed6def9417fd0d44d7
base: main
reviewed_at: 2026-07-26T23:07:09Z
verdict: approve
---

## Summary

Fixes `/ok-to-test` silently no-oping for GitHub Actions workflow runs on same-repo (non-fork) PRs opened by bots (e.g. `github-actions[bot]`). The approve-workflow-run endpoint only works for fork PRs and returns 403 for same-repo bot PRs; previously swallowed as a token-scope warning. On 403 from approve, falls back to the rerun endpoint (`TriggerGitHubWorkflow`), which changes `triggering_actor` and bypasses the approval gate. `approveGitHubActionsWorkflowRuns` now returns a `*sync.WaitGroup` (using Go 1.25's `wg.Go`) so the new test can synchronize on goroutine completion deterministically instead of sleeping. Entire feature is gated behind the pre-existing opt-in `trigger_github_workflows` config flag. ~10 of 14 changed files are unrelated gofmt/whitespace churn plus a stray `plugin-config-documented.yaml` doc-regen addition, confirmed cosmetic/no-op by three independent reviewer passes (code quality, maintainability, deployment risk).

## Findings

### [should-fix] unrelated gofmt/doc churn bloats the diff
- where: 8+ files outside `pkg/plugins/trigger/*` and `pkg/github/fakegithub/fakegithub.go` (e.g. `cmd/deck/main.go`, `cmd/checkconfig/main_test.go`, `pkg/plugins/jira/jira.go`, `pkg/tide/tide_test.go`, `pkg/spyglass/**`, `cmd/external-plugins/cherrypicker/server_test.go`), plus `pkg/plugins/plugin-config-documented.yaml` (+11, `invalid_commit_msg` block)
- concern: Confirmed by three independent reviewers (code-quality, maintainability, deployment-risk) as pure gofmt/whitespace churn and unrelated generated-docs drift (the yaml block documents a pre-existing `InvalidCommitMsg` config struct already in `pkg/plugins/config.go`, not a new feature). Zero behavioral effect, but muddies `git blame`/backport history for files this PR has no functional business touching, and inflates the diff (239+/44- vs. ~40 lines of actual fix). Likely a stale-rebase artifact given the repo's separate "Update gofmt" commit already on `main`.
- excerpt: |
    -	Reporter:           config.Reporter{Context: "ci/prow/test"},
    +	Reporter:            config.Reporter{Context: "ci/prow/test"},

### [nit] discarded WaitGroup return value undocumented at call site
- where: `pkg/plugins/trigger/generic-comment.go:153`
- concern: `approveGitHubActionsWorkflowRuns` now returns `*sync.WaitGroup`, but the only production call site discards it (`approveGitHubActionsWorkflowRuns(c, org, repo, pr.Head.Ref, headSHA)`). Correct behavior (fire-and-forget is intended in production — the WaitGroup exists solely so `TestApproveWorkflowRunsFallback` can `wg.Wait()`), but a one-line comment would prevent a future reader from wondering if the discard is an oversight.

### [nit] pre-existing weak positive-path assertion left unfixed
- where: `pkg/plugins/trigger/generic-comment_test.go:1852-1864`
- concern: `TestApproveGitHubActionsWorkflowRuns`'s `expectApprovalAttempt: true` case still only does `t.Logf(...)` with no real assertion (pre-existing, not introduced here). Now that `wg.Wait()` plumbing exists via the new test, tightening this would be trivial — not blocking since the new `TestApproveWorkflowRunsFallback` covers the added behavior properly.

### [question] 403 fallback scoped broadly
- where: `pkg/plugins/trigger/generic-comment.go:322-332`
- concern: Falls back to rerun on any 403, not only when the PR is confirmed non-fork. A genuine token-scope 403 on a fork PR would also attempt (and likely fail) a rerun instead of clearly surfacing the real permission problem. Soft risk only — the rerun attempt fails safely and logs an error — but the info log ("likely non-fork PR") bakes in an unverified assumption.
- excerpt: |
    } else if github.IsForbidden(err) {
        log.Infof("approve endpoint returned 403 (likely non-fork PR), falling back to rerun: %v", err)

### [question] log level change may silence existing alerts
- where: `pkg/plugins/trigger/generic-comment.go:329`
- concern: 403 case moved from `Warnf("permission denied approving workflow run (check token scopes): %v", err)` to `Infof(...)`. Any operator with alerting keyed to the old warning-level string will stop seeing it. Likely desired (the old message was misleading), but worth a deliberate confirmation.

## Checked
- `TriggerGitHubWorkflow` (`pkg/github/client.go:2240`) correctly hits `POST /repos/{org}/{repo}/actions/runs/{id}/rerun`, matching the PR description.
- `sync.WaitGroup.Go` is valid — go.mod requires go 1.25.8 (method added in Go 1.25).
- Fallback correctly scoped to `IsForbidden` only; `IsNotFound` (race with another approver) and generic errors do not trigger a rerun.
- `TestApproveWorkflowRunsFallback` is thorough: success, 404, 403→rerun-success, 403→rerun-failure, generic-error-no-rerun, multiple runs, mixed success/fallback — all synchronized via `wg.Wait()`, using `sets.New[string]` for order-independent slice comparison.
- `fakegithub.FakeClient` additions (`ApproveWorkflowRunErrors`, `ReranWorkflowRuns`, `ReranWorkflowRunErrors`) follow the existing `map["org/repo/id"]error` convention and reuse the existing lock.
- Entire code path gated behind the pre-existing opt-in `trigger_github_workflows` config flag — no impact on installations not using it.
- No config struct fields renamed/removed, no JSON/YAML tag changes, no new required fields; rollback is safe (no state to unwind); no new GitHub App permissions/scopes required.
- Three independent specialist review passes (code-quality, maintainability, deployment-risk) all converged on APPROVE/LOW-RISK for the functional fix, with the only shared concern being the unrelated churn noted above.

## Open questions
- Can the gofmt/doc-drift-only file changes be dropped or rebased out so this PR stays scoped to the trigger fix?
- Should the 403→rerun fallback be narrowed to cases where the PR is confirmed non-fork (e.g. via `pr.Head.Repo.Fork`) to avoid masking genuine permission errors on fork PRs?
- Is silencing the old `Warnf` "permission denied" log line for this path intentional for any existing log-based alerting?
