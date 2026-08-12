---
pr: kubernetes-sigs/prow#707
title: "approve: fix silent approval bypass when PR exceeds GitHub file list API limit"
head_sha: f8fb3fd1af69203c783876acd791fbf7d757f275
base: main
reviewed_at: 2026-06-30T11:52:48Z
verdict: comment
refresh_log:
  - old_sha: decefa02a730efb3d44e4df79d6bf4810d6a1267
    new_sha: f8fb3fd1af69203c783876acd791fbf7d757f275
    summary: "Force-push removed entire commit-based fallback; getFilenames now returns (nil, true, nil) on truncation with no recovery attempt. Resolves blocking finding. Verdict changed from request-changes to comment."
---

# Review: kubernetes-sigs/prow#707

Multi-perspective maintainer review (code quality, maintainability, deployment risk) + advisor synthesis.

PR summary: GitHub "list PR files" API silently caps at 3000 files; approve plugin evaluated only returned files, so >3000-file PRs could hide unowned files past the cap and still get `approved` (observed: openshift/hypershift#8355). Fix adds `PullRequest.ChangedFiles`, a `getFilenames` helper that compares file count against `ChangedFiles` and returns `(nil, true, nil)` on truncation, and on truncation posts [ApprovalNotifier] NOT APPROVED comment, removes bot-added `approved` label (preserves human-added), returns nil.

Since previous review (`decefa02a` → `f8fb3fd1a`, force-push 2026-06-16):
- Entire commit-based fallback removed: `ListPullRequestCommits`/`GetSingleCommit` calls, `truncationReason` enum, `maxCommitsForFallback` constant, and the union-filenames logic are all gone. `getFilenames` is now ~15 lines returning `([]string, bool, error)`.
- `truncationWarningMessage` simplified to a single message taking only `changedFiles int`.
- Two of three truncation tests deleted (`CommitsFallback`, `StillIncomplete`); the remaining one renamed to `TestHandleTruncatedFileList` and simplified.
- This is a simpler direction than the "option 1" (root approver) discussed on Slack and prototyped on `petr-muller/prow:707-explore`. Current code just blocks entirely on truncation — no root-approver fallback. More conservative but less ergonomic for legitimate large PRs.

Off-PR context (Slack, 2026-06-09): the blocking finding below was independently confirmed on the PR (discussion r3195042137 — GetSingleCommit is also limited). Author (jmguzik) intended to refactor toward "option 1": on truncation, require a top-level (root) approver. The actual push took a simpler approach: just block and require manual admin label. The root-approver direction may come in a follow-up.

Prototype (2026-06-09): reviewer implemented the root-approver direction on petr-muller/prow branch `707-explore` (commit 3e3c66a3c). Design: detection in the github client via typed `PullRequestChangesTruncatedError`; approve catches it and builds root-only approval via `approvers.NewTruncatedOwners`. When/if the author pursues root-approver path, compare against this prototype.

## Findings

### [should-fix] Silent fail-open when ChangedFiles == 0 (pull_request_review path inert)
- where: `pkg/plugins/approve/approve.go:371`, `pkg/plugins/approve/approve.go:275-289` (handleReview)
- concern: `if expectedFiles > 0 && len(filenames) < expectedFiles` — when `expectedFiles` is 0, truncation detection is skipped entirely with no log line. `handleReview` populates `changedFiles` from the `pull_request_review` webhook payload, whose embedded `pull_request` object does not carry `changed_files` — so via review events the check is silently inert and the bypass remains reachable (fails open to pre-PR behavior; gap, not regression). A future handler entry point forgetting to populate `state.changedFiles` would also disable the check with zero signal. Minimum: log when the guard is skipped. Better: fetch PR via `GetPullRequest` when the payload field is 0.
- excerpt: |
    if expectedFiles > 0 && len(filenames) < expectedFiles {
        return nil, true, nil
    }

### [should-fix] Test coverage is thin for a security fix
- where: `pkg/plugins/approve/approve_test.go:1407-1459`
- concern: Only one truncation test remains (`TestHandleTruncatedFileList`). It checks that the truncation warning is posted and the `approved` label is removed, which is good. But still missing: dedup path (second `handle` run with identical existing truncation comment must not delete/recreate), `humanApproved == true` preserving the label in the truncation path, and `expectedFiles == 0` fast-path behavior. All testable with fakegithub today.

### [nit] Unused fetches on truncation path
- where: `pkg/plugins/approve/approve.go:443-450`
- concern: `ListPullRequestComments` and `ListReviews` are fetched before the truncation early-return but unused there (only issueLabels, botUserChecker, issueComments needed). Checking `truncated` earlier saves two paginated calls on the PRs that are already in an exceptional path.

### [nit] Duplicated notification lifecycle logic
- where: `pkg/plugins/approve/approve.go:457-469` vs `pkg/plugins/approve/approve.go:536-550`
- concern: Truncation block re-implements filter-notifications/compare-latest/delete-all/create-new from the normal flow; two copies in one function will drift. Extract a shared helper.

### [question] WasLabelAddedByHuman error handling asymmetry
- where: `pkg/plugins/approve/approve.go:472-476` vs `pkg/plugins/approve/approve.go:574-577`
- concern: On error the truncation path preserves the approved label (fails open on a known-incomplete PR), while existing `humanAddedApproved` assumes not-human on error (fails closed). Deliberate (protect manual admin overrides) or oversight? Align or document the choice.
- excerpt: |
    humanApproved, err := ghc.WasLabelAddedByHuman(pr.org, pr.repo, pr.number, labels.Approved)
    if err != nil {
        log.WithError(err).Errorf("Failed to check ... preserving existing label state.")
        return nil
    }

### [question] Root cause remains for ~15 other GetPullRequestChanges consumers
- where: `pkg/github/client.go:2415` (GetPullRequestChanges)
- concern: Same silent truncation affects blockade (security-relevant: file-based blocks bypassable), verify-owners, owners-label, blunderbuss, trigger, etc. Detection (compare vs `PR.ChangedFiles`) is generic but implemented as approve-private helper. Plan: client-level affordance (typed error / `GetPullRequestChangesChecked`) or at least a tracking issue linked from a code comment?

### [question] Operator-facing documentation
- where: approve plugin docs / helpProvider
- concern: New blocking behavior + "admin must manually apply approved" remediation only documented in the bot comment itself. Update plugin docs/helpProvider? Release-note that bot-applied approved labels on >3000-file PRs get removed on next event after upgrade?

### [question] Direction divergence from discussed plan
- where: overall approach
- concern: Slack discussion indicated "option 1" (root approver on truncation); this push takes the simpler block-entirely approach. Either is valid — just confirming this is the intended final direction or an interim step before root-approver support.

## Resolved

### [blocking] Fallback completeness check is unsound; attacker can re-open the bypass
- where: was `pkg/plugins/approve/approve.go:411-423`
- resolution: Entire commit-based fallback removed. `ListPullRequestCommits`/`GetSingleCommit` no longer called; `getFilenames` detects truncation via `len(filenames) < expectedFiles` and returns immediately. No unsound count-equality heuristic, no per-commit file cap issue.

### [should-fix] Undocumented invariant + maxCommitsForFallback rationale
- where: was `pkg/plugins/approve/approve.go:368, 378-383`
- resolution: `maxCommitsForFallback` constant and all commit-based fallback logic deleted.

### [nit] reason logged as raw iota
- where: was `pkg/plugins/approve/approve.go:518`
- resolution: `truncationReason` enum deleted. Warning log now uses a plain descriptive message.

### [nit] Commit-fetch errors surface with misleading context
- where: was `pkg/plugins/approve/approve.go:398-414`
- resolution: No more commit fetches; `getFilenames` only calls `GetPullRequestChanges`.

### [nit] getFilenames signature at edge of readability
- where: was `pkg/plugins/approve/approve.go:384`
- resolution: Simplified from `([]string, truncationReason, int, error)` to `([]string, bool, error)`.

## Checked
- Fast path byte-identical for <=3000-file PRs: single GetPullRequestChanges call, zero extra API calls; entire pre-existing test table stays on fast path via `ChangedFiles: len(files)` seeding.
- Truncation comment uses approvers.ApprovalNotificationName and matches notificationRegex — participates in existing notification dedup/replace lifecycle; self-cleans when truncation resolves.
- No config/CRD/flag/RBAC/scope changes; `ChangedFiles` struct field is additive (no omitempty — serialized output gains `changed_files: 0`, harmless); rollback is plain image revert; no migration or deploy-ordering. Deployment risk LOW.
- Detection compares against PR.ChangedFiles, not a hardcoded 3000 (3000 appears only in user-facing text) — resilient to GitHub changing the cap.
- Human-applied approved labels never removed; manual-label escape hatch works.
- Returning nil from truncation path keeps hook error metrics clean.
- Tests are behavioral (drive real handle() through fakegithub; no new mocks).
- Transient false-positive window on force-push that shrinks a PR (stale changed_files in payload) self-heals on next synchronize event.
- Change cleanly revertible: one struct field + plugin-local code.
- API cost reduced vs previous revision: no more up-to-11-call fallback; truncation handling adds zero extra API calls beyond the existing GetPullRequestChanges.

## Open questions
- Is the WasLabelAddedByHuman error-path fail-open (preserve label) deliberate, given humanAddedApproved treats the same error fail-closed?
- Plan for the other GetPullRequestChanges consumers (esp. blockade) — client-level fix or tracking issue?
- Should handleReview fetch the PR to populate changed_files, given pull_request_review payloads omit it?
- Docs/helpProvider + release-note updates for the new blocking behavior?
- Is the block-entirely approach the final direction, or is root-approver support (per Slack "option 1") coming as a follow-up?
