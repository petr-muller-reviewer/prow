---
pr: kubernetes-sigs/prow#818
title: "Fix sticky overrides being clobbered by retriggered ProwJobs"
head_sha: 557481b4882edca4dc1b6a458edbfbceb9cbe24a
base: main
reviewed_at: 2026-11-03T22:34:36Z
verdict: approve
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
  - from: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
    to: 557481b4882edca4dc1b6a458edbfbceb9cbe24a
    summary: >-
      Author pushed "Drop ListCheckRuns and preserve retarget buildAll semantics"
      (+29/-52, 3 files). Removed the unconditional ListCheckRuns call from buildAll,
      along with its fakegithub.FakeClient.CheckRuns map/ListCheckRuns method and the
      trigger githubClient interface entry — resolving the remaining should-fix. Also
      added a skipPassing bool threaded through buildAllButDrafts/buildAllIfTrusted/buildAll
      so the PR-retarget path (PullRequestActionEdited with base ref/SHA change) passes
      false and keeps retriggering everything, per the reviewer's retarget comment.
      PR was then APPROVED by petr-muller (2026-08-27T15:00:35Z), labeled lgtm, and
      MERGED (2026-08-27T15:21:50Z). Build + touched-package tests re-verified at new head.
gate:
  decision: hold
  gated_at: 2026-08-18T20:17:29Z
  gated_head_sha: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
  reviewed_head_sha: 82e1f167eb5b7a7a9af5b3ad83b90a27058936f1
---

## Gate

> Superseded: this hold was resolved after the refresh below. Head `557481b48` dropped `ListCheckRuns` entirely and petr-muller approved at 2026-08-27T15:00:35Z; the PR merged at 2026-08-27T15:21:50Z. The text below records the state as of `82e1f167e` and is kept for history.

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

Since previous review (2026-11-03): code changed, `82e1f167e` → `557481b48` ("Drop ListCheckRuns and preserve retarget buildAll semantics", +29/−52 over 3 files):
- `ListCheckRuns` is gone from `buildAll`, from the trigger plugin's `githubClient` interface, and from `fakegithub.FakeClient` (the `CheckRuns` map and its accessor were deleted) — the last open `should-fix` is addressed, and the reviewer's reply "It was some future proofing, agreed that it can be dropped" (2026-08-19T12:21:26Z) confirms it was their ask. Only `GetCombinedStatus` remains, and only when `skipPassing` is true.
- A new `skipPassing bool` is threaded through `buildAllButDrafts`/`buildAllIfTrusted`/`buildAll`. Every caller passes `true` except the retarget path (`PullRequestActionEdited` with a changed base ref or SHA), which passes `false` so a retargeted PR still re-runs all of the new branch's presubmits instead of inheriting a stale same-named passing context — exactly the edge case raised in the 2026-08-18T21:32:42Z review comment.
- Reviewer approved (2026-08-27T15:00:35Z), `lgtm` label added, PR merged 2026-08-27T15:21:50Z.

Since previous review: no code changes. Author (smg247) pinged the reviewer directly: "@petr-muller can you give this another look?" (2026-08-17T18:07:30Z).

## Findings

### [nit] no test exercises buildAll's new skip-on-success wiring directly
- where: `pkg/plugins/trigger/pull-request_test.go` (unchanged)
- concern: `TestAllWithExistingStatusFilter` is well unit-tested in isolation, but `buildAll`'s wiring — including the fallback-to-`TestAllFilter` behavior when `GetCombinedStatus` errors, and the `skipPassing=false` retarget path — still has no direct test coverage; `557481b48` added no tests.

## Resolved (prior review, no longer applicable)

### [should-fix] ListCheckRuns called unconditionally in buildAll, unlike the override plugin's own gating
- resolution: Fixed in `557481b48` — the `ListCheckRuns` call was removed from `buildAll` altogether (along with the `fakegithub.FakeClient.CheckRuns`/`ListCheckRuns` helper and the `githubClient` interface entry), rather than gated. Reviewer confirmed the call was future-proofing and could be dropped (2026-08-19T12:21:26Z).

### [question] buildAll skip-on-success would drop retriggering after a PR retarget
- resolution: Raised by the reviewer on 2026-08-18T21:32:42Z. Fixed in `557481b48`: `skipPassing` is `false` on the `PullRequestActionEdited` base-change path, so retargeted PRs still build everything for the new base branch.

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
- `go build ./pkg/plugins/trigger/... ./pkg/github/...` succeeds at head `557481b48`.
- `go test ./pkg/plugins/trigger/... ./pkg/plugins/override/... ./pkg/pjutil/...` passes at head `557481b48`.
- `git diff 82e1f167e..557481b48` reviewed in full: only the three files above changed; no leftover `ListCheckRuns` references in `pkg/plugins/trigger` or `pkg/github/fakegithub`.
- Every `buildAllIfTrusted`/`buildAllButDrafts` call site checked: all pass `true` except the `PullRequestActionEdited` base-ref/SHA branch, which passes `false`.
- PR was merged 2026-08-27T15:21:50Z (state `MERGED`, labeled `approved` + `lgtm`); no config/API/RBAC breaking changes were ever in scope.
- `/test all` and `/ok-to-test` comments are unaffected by the `buildAll` change (separate code path via `generic-comment.go` / `pjutil.PresubmitFilter`), confirmed independently and by the author.

## Open questions
- The PR merged on 2026-08-27 without the override-plugin abort half. Is there a tracking issue for the deferred `abortJobsForContexts` rework? Still unanswered at merge time — worth filing if none exists, since the merged title still reads as fixing the "clobbered" half, which this PR does not.
- The `skipPassing=false` retarget path added in `557481b48` remains untested (see the `[nit]` above).

## Followups

Accepted 2026-11-03 from post-merge followup on PR #818 (merged as `69388feccbd9d4cdc62ea186db0fd50ac633bf69`). 3 walked, 3 accepted, 0 skipped.

### 1. [tests] Exercise buildAll's skipPassing wiring (Mode: probe)
- where: `pkg/plugins/trigger/pull-request.go:396` (`buildAll`), `:385` (`buildAllButDrafts`), `:228` (`buildAllIfTrusted`); `pkg/plugins/trigger/pull-request_test.go`
- why: this is the one finding left open at merge — `TestTestAllWithExistingStatusFilter` covers the filter in isolation, but nothing drives `buildAll` itself. The retarget branch (`:121`, `skipPassing=false`) is the subtle half of the fix and has a single call site.
- necessity: should

```text
In kubernetes-sigs/prow, following merged PR #818 ("Fix sticky overrides being clobbered by retriggered ProwJobs", merge commit 69388feccbd9d4cdc62ea186db0fd50ac633bf69), probe the trigger plugin's skip-passing wiring for real bugs.

Where: pkg/plugins/trigger/pull-request.go — buildAll (~396), buildAllButDrafts (~385), buildAllIfTrusted (~228), and the call sites at 93, 100, 121, 128, 137, 152, 160, 247. Tests belong in pkg/plugins/trigger/pull-request_test.go, reusing the existing fakegithub-based table harness.

Task: work on current main. Write throwaway tests (uncommitted files in the normal test location) that drive handlePR or buildAll for:
1. PullRequestActionOpened/Reopened with a SUCCESS status on the head SHA whose Context matches one presubmit — that presubmit must not be created, the others must be.
2. GetCombinedStatus returning an error — all presubmits must still be created (wrap fakegithub in a small stub if it cannot be made to error).
3. PullRequestActionEdited with a changed base ref/SHA while a same-named context is already SUCCESS on the head SHA — every presubmit must be created (proves skipPassing=false).
4. PullRequestActionEdited with no base change — no jobs created at all.
Run `go test ./pkg/plugins/trigger/...` and triage every failure: real implementation bug, or wrong expectation/test mistake.

Acceptance: all four behaviours exercised; each failure classified with evidence. If a real bug is found, apply the minimal fix only and report the failing case. If none: leave no files changed.

Scope guard: delete the probe tests before finishing — the deliverable is a bug fix, never the tests. Do not refactor buildAll, do not touch the override plugin, do not fix test-only or style issues, do not push or open a PR.
```

### 2. [tech-debt] Deduplicate the PR ProwJob selector and abort loop (Mode: deliver)
- where: `pkg/plugins/trigger/pull-request.go:171-224` (`abortAllJobs`, `labelSelectorForPR`) vs `pkg/plugins/override/override.go:185-235` (`prowJobSelectorForPR`, `abortActiveJob`, added post-merge in `a3936e14f`)
- why: the deferred override-abort work landed immediately after this merge and re-added a near-verbatim copy of the selector (same org/repo/pull/presubmit label set and empty-selector guard). The two abort loops also diverge: trigger aborts every non-complete presubmit job without clearing `Spec.Report`; override clears `Spec.Report` and filters by `Spec.Job`. This is the "divergent abort implementations" concern the original review raised, now live in main.
- necessity: should

```text
In kubernetes-sigs/prow, following merged PR #818 (merge commit 69388feccbd9d4cdc62ea186db0fd50ac633bf69) and its follow-up commit a3936e14f ("override: abort in-flight jobs so they cannot clobber /override"), consolidate the duplicated PR-scoped ProwJob selector and abort logic.

Current state on main: pkg/plugins/trigger/pull-request.go has labelSelectorForPR (~202) and abortAllJobs (~171); pkg/plugins/override/override.go has prowJobSelectorForPR (~185) and abortActiveJob (~199). The selector bodies are identical (kube.OrgLabel/RepoLabel/PullLabel/ProwJobTypeLabel set plus the klabels empty-selector guard). The abort loops differ: trigger aborts every non-complete presubmit ProwJob for the PR and leaves Spec.Report alone; override sets Spec.Report=false and only aborts jobs whose Spec.Job matches the requested name.

Task: extract one shared selector helper, e.g. PullRequestJobSelector(org, repo string, number int) (klabels.Selector, error), into pkg/kube next to the label constants (or pkg/pjutil if pkg/kube would create an import cycle — check before choosing). Switch both call sites to it. Then unify the abort loop behind one helper with explicit options for (a) clearing Spec.Report and (b) filtering by job name, so trigger's and override's behaviour stay exactly as they are today, including the conflict-tolerant Update and the skip-complete check.

Acceptance: no behavioural change; `go build ./...` and `go test ./pkg/plugins/trigger/... ./pkg/plugins/override/... ./pkg/pjutil/...` pass; only one definition of the selector remains (grep for kube.PullLabel under pkg/).

Scope guard: do not change when or which jobs are aborted, do not touch pjutil.TerminateOlderJobs or other abort paths, no unrelated refactors, do not push or open a PR.
```

### 3. [cleanup] Make the skipPassing argument self-documenting (Mode: deliver)
- where: `pkg/plugins/trigger/pull-request.go:93,100,121,128,137,152,160,247` (call sites), `:228`, `:385`, `:396` (signatures)
- why: the fix threads a bare bool through three functions, so eight call sites read `..., true)` / `..., false)`, and only the `PullRequestActionEdited` retarget branch at `:121` is the inverse of the rest — the easiest place for a future edit to silently reintroduce the bug. Behaviour is correct today, so it did not block the merge.
- necessity: could

```text
In kubernetes-sigs/prow, following merged PR #818 (merge commit 69388feccbd9d4cdc62ea186db0fd50ac633bf69), make the trigger plugin's skipPassing argument self-documenting.

Where: pkg/plugins/trigger/pull-request.go — signatures of buildAll (~396), buildAllButDrafts (~385), buildAllIfTrusted (~228), and their call sites at lines 93, 100, 121, 128, 137, 152, 160, 247. All pass true except line 121 (PullRequestActionEdited with a changed base ref/SHA), which passes false so a retargeted PR rebuilds every presubmit instead of inheriting a same-named passing context from the old base branch.

Task: replace the bare bool with a named boolean type (e.g. type passingPolicy bool with constants skipPassing and rerunAll — pick names that read naturally at the call sites) or a small options struct if that is cleaner across all eight sites. Convert the signatures and every call site. Update the doc comments on buildAllIfTrusted, buildAllButDrafts and buildAll to state that the retarget path reruns everything and why.

Acceptance: `go build ./...` and `go test ./pkg/plugins/trigger/...` pass; no call site passes an unlabelled bool; behaviour is unchanged; reading the retarget call site makes the exception obvious without consulting the callee.

Scope guard: no functional change and no new tests (a separate probe covers the wiring), do not touch override or pjutil, do not push or open a PR.
```
