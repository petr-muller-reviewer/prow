---
pr: kubernetes-sigs/prow#798
title: "fix(trigger): fall back to rerun when approve fails for non-fork PRs"
head_sha: dd76a90690dded89117e44ed6def9417fd0d44d7
base: main
reviewed_at: 2026-10-25T13:25:32Z
verdict: approve
refresh_log:
  - at: 2026-10-25T13:25:32Z
    old_sha: dd76a90690dded89117e44ed6def9417fd0d44d7
    new_sha: dd76a90690dded89117e44ed6def9417fd0d44d7
    summary: "No code changes; recorded approval activity and the PR's merge on 2026-08-26."
---

## What this PR does

Fixes `/ok-to-test` silently no-oping for GitHub Actions workflow runs on same-repo (non-fork) PRs opened by bots (e.g. `github-actions[bot]`). The approve-workflow-run endpoint only works for fork PRs and returns 403 for same-repo bot PRs; previously swallowed as a token-scope warning. On 403 from approve, falls back to the rerun endpoint (`TriggerGitHubWorkflow`), which changes `triggering_actor` and bypasses the approval gate. `approveGitHubActionsWorkflowRuns` now returns a `*sync.WaitGroup` (using Go 1.25's `wg.Go`) so the new test can synchronize on goroutine completion deterministically instead of sleeping. Entire feature is gated behind the pre-existing opt-in `trigger_github_workflows` config flag. ~10 of 14 changed files are unrelated gofmt/whitespace churn plus a stray `plugin-config-documented.yaml` doc-regen addition, confirmed cosmetic/no-op by three independent reviewer passes (code quality, maintainability, deployment risk).

Since previous review:
- No code changes: the PR head remains `dd76a90690dded89117e44ed6def9417fd0d44d7`.
- Prucek submitted an approval on 2026-08-10; cblecker approved via `/approve` on 2026-08-26, and the PR was merged on 2026-08-26 at 17:03:39 UTC.

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

## Followups

Post-merge followup to PR #798, merged on 2026-08-26 as `1daa92775867c044e465e11efb6b6269c3e7618a`. These are handoffs for work from the current default branch; no implementation was performed during this walkthrough.

Accepted: 3 (one probe, two deliverables). Skipped: 1 (documenting the intentional WaitGroup discard).

The unrelated formatting/generated-document churn is retained as review history: reversing already-merged cosmetic changes would add churn without improving behavior. The two 403/logging questions are combined in the permission-handling handoff below.

### 1. Probe the /ok-to-test workflow integration and configuration gates

Category: tests. Necessity: should.

```text
Mode: probe

In kubernetes-sigs/prow, following merged PR #798 — "fix(trigger): fall back to rerun when approve fails for non-fork PRs" (merge commit 1daa92775867c044e465e11efb6b6269c3e7618a), probe the command-to-workflow integration in pkg/plugins/trigger/generic-comment.go and TestApproveGitHubActionsWorkflowRuns in pkg/plugins/trigger/generic-comment_test.go. Work from the current default branch and first check whether this gap has already been addressed.

Task: Write temporary tests that exercise handleGenericComment with a trusted /ok-to-test commenter, trigger_github_workflows enabled, and pending workflow runs for the exact PR head. Verify exact run IDs for successful approval, 403-to-rerun fallback, and multiple pending runs. Verify that disabling the flag, enabling IgnoreOkToTest, or using /test all or /retest does not invoke the pending-approval path. Distinguish that approval path from the separate failed-workflow retry path. Use a narrow observing fake or wrapper with channels/synchronization to await asynchronous completion; avoid arbitrary sleeps and unsynchronized reads of fake-client slices. Keep production asynchronous behavior intact.

Acceptance criteria: Exercise every named behavior and classify each failure as an implementation bug or a wrong test expectation. A real bug means that an eligible command fails to approve/rerun the intended run, targets the wrong PR head/run, or invokes pending approval despite a disabled command/configuration gate. Apply the minimal fix only for confirmed implementation bugs. Run the focused probe tests, including the race detector where practical. Delete all probe tests and scaffolding before finishing; if no real bug is found, leave no files changed. Report the cases exercised and the evidence for any bug found.

Scope guard: Do not submit the probe tests, refactor the workflow helper, change the 403 policy, change GitHub API semantics, or alter public handler interfaces merely to enable testing. Do not push, create issues, or open a PR.
```

### 2. Distinguish fork permission errors from same-repo approval fallback

Category: deferred-review. Necessity: could.

```text
Mode: deliver

In kubernetes-sigs/prow, following merged PR #798 — "fix(trigger): fall back to rerun when approve fails for non-fork PRs" (merge commit 1daa92775867c044e465e11efb6b6269c3e7618a), clarify 403 handling in pkg/plugins/trigger/generic-comment.go, with focused coverage in generic-comment_test.go. Work from the current default branch and re-check the code before changing it.

Task: Classify the PR as confirmed same-repo, confirmed fork, or unknown by comparing populated head/base repository identities. Do not use the head repository's Fork flag alone: a PR within a repository that is itself a fork is still a same-repo PR. Carry the classification into approveGitHubActionsWorkflowRuns. On an approval 403, preserve the rerun fallback for confirmed same-repo PRs; for confirmed fork PRs, log a permission-denied warning and do not rerun. When metadata is insufficient, preserve the existing fallback behavior and use neutral wording rather than claiming the PR is likely non-fork. If fallback also fails, emit a diagnostic containing both the original approval error and the rerun error, retaining run/repository context. Keep successful fallback at informational level.

Acceptance criteria: Focused tests cover same-repo 403 -> rerun, fork 403 -> warning/no rerun, unknown metadata 403 -> existing fallback, and failed fallback diagnostics. Account for the same-repo-within-a-fork case. Preserve success, 404, and non-403 behavior, the trigger_github_workflows opt-in gate, and asynchronous production handling. Run the focused trigger tests with deterministic synchronization.

Scope guard: Limit changes to PR classification, approval fallback decisions, diagnostics, and directly related tests. Add no configuration flags or dependencies, do not change workflow discovery or REST endpoint implementations, and do not revert cosmetic changes from PR #798. Do not push, create issues, or open a PR.
```

### 3. Document GitHub Actions approval and rerun behavior in trigger help

Category: docs. Necessity: could.

```text
Mode: deliver

In kubernetes-sigs/prow, following merged PR #798 — "fix(trigger): fall back to rerun when approve fails for non-fork PRs" (merge commit 1daa92775867c044e465e11efb6b6269c3e7618a), document the GitHub Actions integration in pkg/plugins/trigger/trigger.go's helpProvider and the TriggerGitHubWorkflows comment in pkg/plugins/config.go. Work from the current default branch and inspect its actual behavior first; a separate accepted followup may have refined fork-versus-same-repo 403 handling.

Task: Extend the existing trigger plugin help to explain that, when trigger_github_workflows is enabled and /ok-to-test is honored for a trusted command, /ok-to-test attempts to approve pending GitHub Actions runs for the PR head and may rerun after approval returns 403, according to the current implementation. Distinguish this from /retest and /test all retrying failed workflows. Explain that workflow API work completes asynchronously and that failures are reported in hook/plugin logs. Expand the configuration field comment accordingly, then regenerate pkg/plugins/plugin-config-documented.yaml using the repository's documented generator.

Acceptance criteria: Both command help and configuration reference describe the opt-in gate, pending approval, applicable 403 fallback, and failure reporting accurately. Use wording that reflects current fork/same-repo handling and does not promise that rerunning resolves all permission errors or changes the original actor's privileges. Generated documentation agrees with the source comment; inspect the generated diff for unrelated drift. Run only the relevant documentation/help checks required by the repository.

Scope guard: Update the existing help and configuration documentation, plus directly related help expectations if required. Do not change production behavior, add a new flag or a separate documentation subsystem, or include unrelated formatting/generated-document drift. Do not push, create issues, or open a PR.
```

