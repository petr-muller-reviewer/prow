---
pr: kubernetes-sigs/prow#707
title: "approve: fix silent approval bypass when PR exceeds GitHub file list API limit"
head_sha: f8fb3fd1af69203c783876acd791fbf7d757f275
base: main
reviewed_at: 2026-08-12T11:22:19Z
verdict: request-changes
refresh_log:
  - old_sha: decefa02a730efb3d44e4df79d6bf4810d6a1267
    new_sha: f8fb3fd1af69203c783876acd791fbf7d757f275
    summary: "Force-push removed entire commit-based fallback; getFilenames now returns (nil, true, nil) on truncation with no recovery attempt. Resolves blocking finding. Verdict changed from request-changes to comment."
  - old_sha: f8fb3fd1af69203c783876acd791fbf7d757f275
    new_sha: f8fb3fd1af69203c783876acd791fbf7d757f275
    summary: "No code change. Independent /code-review pass (medium) confirmed the handleReview gap against repo fixture review_approve_submitted.json (no changed_files key) — promoted to blocking. Added: truncation detection false-positives on any count shortfall; warning message asserts unverified 3000 claim; notification churn from count-sensitive dedup; fakegithub client coupling. Verdict comment -> request-changes."
  - old_sha: f8fb3fd1af69203c783876acd791fbf7d757f275
    new_sha: f8fb3fd1af69203c783876acd791fbf7d757f275
    summary: "No code change. Cross-checked review #4463008939 (petr-muller, 2026-06-09) three-point plan against shipped code; expanded direction-divergence finding with the concrete gaps."
---

# Review: kubernetes-sigs/prow#707

Multi-perspective maintainer review (code quality, maintainability, deployment risk) + advisor synthesis, plus an independent `/code-review` pass at medium effort.

PR summary: GitHub "list PR files" API silently caps at 3000 files; approve plugin evaluated only returned files, so >3000-file PRs could hide unowned files past the cap and still get `approved` (observed: openshift/hypershift#8355). Fix adds `PullRequest.ChangedFiles`, a `getFilenames` helper that compares file count against `ChangedFiles` and returns `(nil, true, nil)` on truncation, and on truncation posts [ApprovalNotifier] NOT APPROVED comment, removes bot-added `approved` label (preserves human-added), returns nil.

Since previous review (`decefa02a` → `f8fb3fd1a`, force-push 2026-06-16):
- Entire commit-based fallback removed: `ListPullRequestCommits`/`GetSingleCommit` calls, `truncationReason` enum, `maxCommitsForFallback` constant, and the union-filenames logic are all gone. `getFilenames` is now ~15 lines returning `([]string, bool, error)`.
- `truncationWarningMessage` simplified to a single message taking only `changedFiles int`.
- Two of three truncation tests deleted (`CommitsFallback`, `StillIncomplete`); the remaining one renamed to `TestHandleTruncatedFileList` and simplified.
- This is a simpler direction than the "option 1" (root approver) discussed on Slack and prototyped on `petr-muller/prow:707-explore`. Current code just blocks entirely on truncation — no root-approver fallback. More conservative but less ergonomic for legitimate large PRs.

Since previous refresh (2026-08-11, no code change): independent `/code-review` pass confirmed the `handleReview` gap with fixture evidence and surfaced four additional findings. `go test ./pkg/plugins/approve/...` passes on `f8fb3fd1a`.

Off-PR context (Slack, 2026-06-09): the original blocking finding was independently confirmed on the PR (discussion r3195042137 — GetSingleCommit is also limited). Author (jmguzik) intended to refactor toward "option 1": on truncation, require a top-level (root) approver. The actual push took a simpler approach: just block and require manual admin label. The root-approver direction may come in a follow-up.

Prototype (2026-06-09): reviewer implemented the root-approver direction on petr-muller/prow branch `707-explore` (commit 3e3c66a3c). Design: detection in the github client via typed `PullRequestChangesTruncatedError`; approve catches it and builds root-only approval via `approvers.NewTruncatedOwners`. When/if the author pursues root-approver path, compare against this prototype.

## Findings

### [blocking] The pull_request_review path is inert — the bypass stays open on that entry point
- where: `pkg/plugins/approve/approve.go:284`, `pkg/plugins/approve/approve.go:378`
- concern: `handleReview` seeds `changedFiles` from `re.PullRequest.ChangedFiles`, and GitHub's reduced PR object in the `pull_request_review` payload carries no `changed_files`. Confirmed against the repo's own fixture: `cmd/phony/examples/review_approve_submitted.json` has zero occurrences of `changed_files`, while `opened_pr.json` has one. So `pr.changedFiles == 0`, the `expectedFiles > 0` guard short-circuits, and `truncated` is always false on that path. Concrete scenario: repo sets `ignore_review_state: false`; an approver submits an APPROVE review on a 5000-file PR; `handle` runs with the truncated file list and applies `approved` — exactly the bypass this PR exists to close. `handleGenericComment` (line 210, fed by `GetPullRequest`) and `handlePullRequest` (line 343, full PR in payload) are fine. Fix: `handleReview` needs its own `ghc.GetPullRequest` call, or the guard needs to treat 0 as unknown rather than as not-truncated.
- excerpt: |
    changedFiles: re.PullRequest.ChangedFiles,
    ...
    if expectedFiles > 0 && len(filenames) < expectedFiles {
        return nil, true, nil
    }

### [should-fix] Truncation detection false-positives on any count shortfall, not just the 3000 cap
- where: `pkg/plugins/approve/approve.go:378`
- concern: `expectedFiles` is a snapshot (webhook payload, or a `GetPullRequest` issued before the `GetPullRequestChanges` call), so it can legitimately exceed the live file count. Scenario: author pushes commit A (100 files), force-pushes B (5 files) seconds later; hook processes the `synchronize` event for A after B landed → `5 < 100` → `truncated == true` on a 5-file PR. The plugin then refuses to compute approval, posts the "exceeds the GitHub API limit of 3000 files" comment, and strips `approved`. Gating on `len(filenames) >= 3000` as well would confine the branch to actual cap hits. Previously logged under Checked as a self-healing transient; it does self-heal on the next event, but the user-visible cost (wrong comment + label removal) makes it a finding.
- excerpt: |
    if expectedFiles > 0 && len(filenames) < expectedFiles {
        return nil, true, nil
    }

### [should-fix] Warning message asserts a fact the code never verified
- where: `pkg/plugins/approve/approve.go:384-391`
- concern: The message hardcodes "changes %d files, which exceeds the GitHub API limit of 3000 files" with `pr.changedFiles` interpolated, but the branch is reachable at any count (see previous finding). Anyone hitting the force-push race gets a comment claiming their 100-file PR exceeds 3000 files, plus an unexplained label removal. Either gate the branch on the real cap or word the message so it does not assert a threshold the code did not check.

### [should-fix] WasLabelAddedByHuman error path fails open on the security-relevant branch
- where: `pkg/plugins/approve/approve.go:472-476` vs `pkg/plugins/approve/approve.go:574-577`
- concern: On error the truncation path logs and `return nil`, leaving a bot-applied `approved` label in place on a PR whose OWNERS coverage is admittedly unknown — the stale approval the commit message says it prevents. A transient GitHub 5xx is enough, and nothing retries, so the label survives until the next event. The normal path's `humanAddedApproved` treats the same error as "not human-approved" (fails closed). Was a [question] in the previous review; the asymmetry lands on the unsafe side, so raising it. Align the two, or document why preserving the label wins here.
- excerpt: |
    humanApproved, err := ghc.WasLabelAddedByHuman(pr.org, pr.repo, pr.number, labels.Approved)
    if err != nil {
        log.WithError(err).Errorf("Failed to check ... preserving existing label state.")
        return nil
    }

### [should-fix] Test coverage is thin for a security fix
- where: `pkg/plugins/approve/approve_test.go:1405-1459`
- concern: Only one truncation test remains (`TestHandleTruncatedFileList`), covering the truncated direction only. Every pre-existing `TestHandle` case constructs `state{}` without `changedFiles`, so they all run with `expectedFiles == 0` and skip the new check entirely — no case asserts that normal approval still happens with `changedFiles: len(files)`, which is what would catch an off-by-one or inverted comparison. Also missing: dedup path (second `handle` run with identical existing truncation comment must not delete/recreate) and `humanApproved == true` preserving the label. Note the `fgc.PullRequests` entry added in `newFakeGitHubClient` (line 80) is never consumed — `handle` does not call `GetPullRequest`.

### [nit] Notification churn on large PRs
- where: `pkg/plugins/approve/approve.go:456-470`
- concern: `msg` embeds `pr.changedFiles`, so the "is this comment already posted" comparison (`!strings.Contains(latestNotification.Body, msg)`) is count-sensitive. Every push that changes the file count (5001 → 5002) makes bodies differ, deleting all prior notifications and posting a fresh one — a new email per push on precisely the giant PRs this targets. Matching on a count-free prefix avoids it. Separately, the delete-then-create ordering means a failed `CreateComment` leaves the PR with no approval notification at all and only a log line.

### [nit] Unused fetches on truncation path
- where: `pkg/plugins/approve/approve.go:443-450`
- concern: `ListPullRequestComments` and `ListReviews` are fetched before the truncation early-return but unused there (only issueLabels, botUserChecker, issueComments needed). Checking `truncated` earlier saves two paginated calls on the PRs that are already in an exceptional path.

### [nit] Duplicated notification lifecycle logic
- where: `pkg/plugins/approve/approve.go:456-470` vs `pkg/plugins/approve/approve.go:536-550`
- concern: Truncation block re-implements filter-notifications/compare-latest/delete-all/create-new from the normal flow; two copies in one function will drift. Extract a shared helper.

### [nit] fakegithub client now reports every PR as truncated
- where: `pkg/github/client.go:2502-2504`
- concern: `GetPullRequestChanges` returns an empty slice when `c.fake` is set, so any code path built on `github.NewFakeClient()` now sees `0 < changed_files` and reports truncation. Does not appear reachable from production `hook` wiring, but it is a new behavioral coupling between the fake client and the plugin worth knowing about.
- excerpt: |
    if c.fake {
        return []PullRequestChange{}, nil
    }

### [question] Root cause remains for ~15 other GetPullRequestChanges consumers
- where: `pkg/github/client.go:2498` (GetPullRequestChanges)
- concern: Same silent truncation affects blockade (security-relevant: file-based blocks bypassable), verify-owners, owners-label, blunderbuss, trigger, etc. Detection (compare vs `PR.ChangedFiles`) is generic but implemented as approve-private helper. Plan: client-level affordance (typed error / `GetPullRequestChangesChecked`) or at least a tracking issue linked from a code comment?

### [question] Operator-facing documentation
- where: approve plugin docs / helpProvider
- concern: New blocking behavior + "admin must manually apply approved" remediation only documented in the bot comment itself. Update plugin docs/helpProvider? Release-note that bot-applied approved labels on >3000-file PRs get removed on next event after upgrade?

### [question] Direction divergence from discussed plan
- where: overall approach
- concern: Review #4463008939 (petr-muller, 2026-06-09, on commit `decefa02a`) proposed a specific three-part plan; the shipped code diverges on all three points:
  1. "Make `GetPullRequestChanges` detect the case and return error... any caller will need to handle the case appropriately." Not done — `pkg/github/client.go:2499` `GetPullRequestChanges` is unchanged, no crosscheck, no error. Detection lives privately in the `approve` plugin's `getFilenames` instead, so the other ~15 callers of `GetPullRequestChanges` (blockade, verify-owners, owners-label, blunderbuss, trigger, ...) remain exposed to the same silent truncation.
  2. "Any caller is advised to do `GetPullRequest` first... to avoid calling potentially paginated `GetPullRequestChanges` that's doomed to fail" (i.e. pre-check the count, skip the paginated call above the threshold). Not implemented — the plugin always calls `GetPullRequestChanges` first and compares after the fact, and `handleReview` doesn't call `GetPullRequest` at all (see blocking finding).
  3. "The `approve` plugin would fall back to root-level approver instead of handling the actual paths as normal." Not implemented — shipped code fully blocks on truncation (NOT APPROVED comment, strip bot-applied label, require manual admin re-approval), no root-approver fallback logic exists in this PR. That direction only exists in reviewer's prototype `petr-muller/prow:707-explore` (commit `3e3c66a3c`).
  Also moot: the review's ask to handle `GetSingleCommit` similarly no longer applies since the entire commit-based fallback that used it was deleted rather than fixed.
  Net: a materially simpler, more conservative implementation (detect-and-block, plugin-local) than what was requested (detect-and-degrade-to-root-approver, client-level). Confirming whether this is accepted as the final direction or an interim step.

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
- `go test ./pkg/plugins/approve/...` passes on `f8fb3fd1a`.
- Fast path byte-identical for <=3000-file PRs: single GetPullRequestChanges call, zero extra API calls; entire pre-existing test table stays on fast path (all cases leave `changedFiles` zero).
- Truncation comment uses approvers.ApprovalNotificationName and matches notificationRegex — participates in existing notification dedup/replace lifecycle; self-cleans when truncation resolves. (Dedup is count-sensitive — see churn nit.)
- No config/CRD/flag/RBAC/scope changes; `ChangedFiles` struct field is additive (no omitempty — serialized output gains `changed_files: 0`, harmless); rollback is plain image revert; no migration or deploy-ordering. Deployment risk LOW.
- Detection compares against PR.ChangedFiles, not a hardcoded 3000 (3000 appears only in user-facing text) — resilient to GitHub changing the cap, though this is also what makes the shortfall check over-trigger.
- Human-applied approved labels never removed (when `WasLabelAddedByHuman` succeeds); manual-label escape hatch works.
- Returning nil from truncation path keeps hook error metrics clean.
- Tests are behavioral (drive real handle() through fakegithub; no new mocks).
- Change cleanly revertible: one struct field + plugin-local code.
- API cost reduced vs previous revision: no more up-to-11-call fallback; truncation handling adds zero extra API calls beyond the existing GetPullRequestChanges.
- `handleGenericComment` (line 210) and `handlePullRequest` (line 343) both populate `changedFiles` from sources that actually carry it — only `handleReview` is affected.

## Open questions
- Will `handleReview` fetch the PR to populate `changed_files`? Without it the `pull_request_review` entry point still has the bypass the PR is closing.
- Should the truncation branch require `len(filenames) >= 3000` so a stale `changed_files` snapshot cannot trigger the "exceeds 3000 files" comment on a small PR?
- Is the WasLabelAddedByHuman error-path fail-open (preserve label) deliberate, given humanAddedApproved treats the same error fail-closed?
- Plan for the other GetPullRequestChanges consumers (esp. blockade) — client-level fix or tracking issue?
- Docs/helpProvider + release-note updates for the new blocking behavior?
- Is the block-entirely approach the final direction, or is root-approver support (per Slack "option 1") coming as a follow-up?
