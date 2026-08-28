---
pr: kubernetes-sigs/prow#818
title: "Fix sticky overrides being clobbered by retriggered ProwJobs"
head_sha: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
base: main
reviewed_at: 2026-08-18T20:03:13Z
verdict: needs-discussion
refresh_log:
  - from: bc324a6d6d8c65c23cf73635570b5afb46d416f7
    to: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
    summary: >-
      Author removed the entire override-plugin abort path (abortJobsForContexts and its
      tests, prowJobClient.List/Update, all related imports) in response to the two blocking
      findings below, deferring that fix to a separate follow-up PR. Only the buildAll
      skip-on-success fix (trigger plugin) remains in scope; its doc comment was reworded to
      be less sticky-override-specific. No functional change to buildAll itself. Worktree
      synced to this head via git reset --hard; build and touched-package tests re-verified.
  - from: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
    to: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
    summary: >-
      No code changes. Author (smg247) replied to the outstanding "still want to dig into
      this a bit more" review thread, confirming buildAll's scope concern is settled:
      /test all uses TestAllFilter (only filters NeedsExplicitTrigger), unaffected by this
      change. Closes the previously open question on buildAll's scope.
  - from: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
    to: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
    summary: >-
      No code changes. Author posted "/hold cancel" (2026-08-03T11:29:41Z); PR carries no
      hold label. PR is already labeled "approved".
  - from: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
    to: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
    summary: >-
      No code changes. Reviewer (petr-muller) posted a "/cc" comment (2026-08-08T23:40:30Z);
      no new reviews, labels, or discussion. PR remains OPEN and "approved".
  - from: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
    to: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
    summary: >-
      No code changes. Author (smg247) pinged the reviewer directly: "@petr-muller can you
      give this another look?" (2026-08-17T18:07:30Z). No new reviews or label changes.
gate:
  decision: hold
  gated_at: 2026-08-18T20:17:29Z
  gated_head_sha: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
  reviewed_head_sha: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
---

## Gate

**Decision: hold** — gated at `82e1f167e` (2026-08-18T20:17:29Z), same head the review covered.

The scope reduction (dropping the override-abort path) resolved both original blocking findings and the open scope question, and the author's `/hold cancel` reflects that. Nothing in this PR is backward-incompatible or risky to existing deployments: `ListCheckRuns` is already part of production `pkg/github.Client`, so adding it to the trigger plugin's local interface subset is not an API break, and `buildAll`'s skip-on-success behavior only affects PR-open/reopen/LGTM/ok-to-test flows (confirmed independently and by the author not to touch `/test all`/`/ok-to-test`). What's missing is a close-out: the one remaining `should-fix` (unconditional `ListCheckRuns` call, an avoidable API-rate-limit cost for installations without GitHub Apps/Checks) has never been addressed or explicitly waved off by a reviewer, and no reviewer (including me) has posted an actual approval/LGTM on the current, narrowed scope — the last substantive reviewer input was a "still want to dig into this" comment that got answered by the author, not a sign-off. The author's 2026-08-17 ping is effectively asking for that sign-off. Recommend: either fix the `ListCheckRuns` gating (cheap, low-risk) or have the reviewer explicitly accept it as-is, then approve.

Gating list:
- `pkg/plugins/trigger/pull-request.go:411-420` — `[should-fix]` `ListCheckRuns` called unconditionally in `buildAll` (from `REVIEW.md`); still unaddressed in current head. Not blocking on its own (no functional bug), but nothing has resolved or explicitly accepted it yet.
- No reviewer approval/LGTM has been posted against the current (post-scope-reduction) diff — the last reviewer engagement was a clarifying question, now answered by the author but not followed by a sign-off.

Area 2 (independent merge risk): no notable risk found. `ListCheckRuns` addition to `pkg/plugins/trigger`'s local `githubClient` interface is additive only and already implemented by production `pkg/github.Client` and `fakegithub.FakeClient` — no breakage for existing deployments. `buildAll`'s new skip-on-success filtering changes behavior only for contexts already reported `SUCCESS`/`neutral` on the PR head SHA at PR-open/reopen/label-trigger time; this is the intended fix and matches the PR's stated purpose, not an unannounced behavior change elsewhere.

## What this PR does

- Trigger plugin: `buildAll` (`pkg/plugins/trigger/pull-request.go`) fetches combined status + check runs for the PR head SHA and skips presubmits whose context is already `SUCCESS`, via `pjutil.TestAllWithExistingStatusFilter`. Prevents e.g. an untrusted-PR LGTM from re-triggering and clobbering an already-overridden/passing context.
- Adds `ListCheckRuns` to `fakegithub.FakeClient` and to the trigger plugin's `githubClient` interface (production `pkg/github.Client` already implements it — no real-deployment gap).
- Unit tests added for `pjutil.TestAllWithExistingStatusFilter`.

**History:**
- The override-plugin abort fix (`abortJobsForContexts`, the source of both original blocking findings) was removed entirely in commit `82e1f167e` ("Skip already-passing presubmits in buildAll"), along with ~150 lines of tests and the `prowJobClient.List`/`Update` interface additions. PR title narrowed from "...clobbered by completing/retriggered ProwJobs" to "...clobbered by retriggered ProwJobs", matching the reduced scope.
- Author (smg247) confirmed this was intentional: "I will re-work this part in a separate PR. I think the `trigger` issue is the more important one to solve" (2026-07-30T19:46:02Z), and separately confirmed the `buildAll` change does not affect `/test all` (matches this review's own finding): "`/test all` ... never calls `buildAll`. `buildAll` is only called from the PR event handler" (2026-07-30T19:48:27Z).
- A later review comment (2026-07-30T21:38:36Z, same thread on `buildAll`'s doc comment) had stated "I still want to dig into this a bit more to make sure I'm not overlooking something..." — since resolved (see below).

Since previous review: no code changes. The author replied to the outstanding review thread (2026-07-31T14:05:51Z), confirming the `buildAll` scope concern is settled: "looks like the above is correct. It uses the `TestAllFilter` which only filters out tests that match `NeedsExplicitTrigger`. It is unaffected by this change." This closes the previously open question on `buildAll`'s scope.

Since previous review: no code changes. Author posted `/hold cancel` (2026-08-03T11:29:41Z); the PR carries no hold label and is labeled `approved`.

Since previous review: no code changes. Reviewer (petr-muller) posted a `/cc` comment (2026-08-08T23:40:30Z); no new reviews, labels, or discussion followed.

Since previous review: no code changes. Author (smg247) pinged the reviewer directly: "@petr-muller can you give this another look?" (2026-08-17T18:07:30Z).

## Findings

### [should-fix] ListCheckRuns called unconditionally in buildAll, unlike the override plugin's own gating
- where: `pkg/plugins/trigger/pull-request.go:411-420`
- concern: The (now-removed) override plugin abort logic gated its own `ListCheckRuns` call behind `oc.UsesAppAuth()`. `buildAll`'s call remains unconditional, adding an extra GitHub API call to every PR-opened/reopened/LGTM/ok-to-test-label event even for installations that don't use GitHub Apps/Checks (where it always returns empty). Not harmful, but avoidable rate-limit cost at scale.
- excerpt: |
    checkRunList, err := c.GitHubClient.ListCheckRuns(org, repo, pr.Head.SHA)
    if err != nil {
        c.Logger.WithError(err).Warn("Failed to list check runs; some passing checks may be re-triggered")
    }

### [nit] no test exercises buildAll's new skip-on-success wiring directly
- where: `pkg/plugins/trigger/pull-request_test.go` (unchanged)
- concern: `TestAllWithExistingStatusFilter` is well unit-tested in isolation, but `buildAll`'s wiring — including the fallback-to-`TestAllFilter` behavior when `GetCombinedStatus`/`ListCheckRuns` error — has no direct test coverage.

## Resolved (prior review, no longer applicable)

### [blocking] abortJobsForContexts marked jobs Complete(), hiding them from plank and leaking the running pod
- resolution: Entire `abortJobsForContexts` function and its call sites removed in `82e1f167e`. Author intends to rework in a separate PR.

### [blocking] missing PrevReportStates bookkeeping could let crier re-clobber the sticky status
- resolution: Same removal as above — the code path that would have raced with crier no longer exists in this PR.

### [should-fix] conflict on Update was logged as a successful abort
- resolution: Removed along with `abortJobsForContexts`.

### [should-fix] abort failures were silently swallowed, not surfaced to the PR comment
- resolution: Removed along with `abortJobsForContexts`.

### [should-fix] three divergent "abort a ProwJob" implementations, no consolidation
- resolution: The third implementation (`abortJobsForContexts`) no longer exists in this PR. Worth re-raising when the deferred override-abort follow-up PR appears — `abortAllJobs` (trigger) and `TerminateOlderJobs` (pjutil) still duplicate similar logic between themselves.

### [nit] override_test.go asserted CompletionTime is set on abort, encoding the problematic behavior
- resolution: The entire `TestHandleStickyOverride` abort-related test cases and `makeProwJob` helper were removed along with the production code.

### [nit] /override-sticky help text not updated
- resolution: Moot — no abort behavior remains in this PR to document.

### [question] scope of buildAll's skip-if-already-successful beyond LGTM
- resolution: Fully answered. Author confirmed `/test all`/`/ok-to-test` comments are unaffected (matches this review's independent finding), and on 2026-07-31T14:05:51Z confirmed the broader scope concern raised on 2026-07-30T21:38:36Z is settled: `/test all` uses `TestAllFilter`, which only filters on `NeedsExplicitTrigger` and is unaffected by this change.

## Checked
- `go build ./...` succeeds at head `82e1f167e`.
- `go test ./pkg/plugins/override/... ./pkg/pjutil/... ./pkg/plugins/trigger/...` passes at head `82e1f167e`.
- `git diff bc324a6d6..82e1f167e -- pkg/plugins/override/override.go` confirmed clean, complete removal of `abortJobsForContexts`, the `sticky` abort-trigger block in `handle()`, and the `List`/`Update` additions to `prowJobClient` — no leftover fragments.
- `buildAll` in `pkg/plugins/trigger/pull-request.go` is functionally unchanged from the original review; only its doc comment was reworded (less sticky-override-specific wording).
- PR is still OPEN; no merge/close.
- No config/API/RBAC breaking changes remain in scope; production `pkg/github.Client` already implements `ListCheckRuns`.
- `/test all` and `/ok-to-test` comments are unaffected by the `buildAll` change (separate code path via `generic-comment.go` / `pjutil.PresubmitFilter`), confirmed independently and by the author.

## Open questions
- Is there a tracking issue/PR yet for the deferred override-plugin abort rework the author mentioned?
- Given the override-abort fix is now out of scope, should the PR description be updated further to avoid implying the "completing ProwJobs" half of the original bug is fixed here?
