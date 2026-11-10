---
pr: kubernetes-sigs/prow#951
title: "Map scheduling ProwJob state to GitHub pending status"
head_sha: 3e50f77192258c8513d2664cbd103051b6b8d272
base: main
reviewed_at: 2026-09-21T23:45:23Z
verdict: approve
---

## Verdict

Approve. This small, localized fix maps the previously unhandled scheduling ProwJob state to the existing GitHub `pending` status and covers the externally visible status-reporting behavior. It makes no configuration, API, dependency, or permission changes.

## What this PR does

- Treats `prowapi.SchedulingState` as GitHub `pending` in the GitHub reporter.
- Keeps scheduled-but-not-yet-running jobs in the same non-terminal status category as triggered and pending jobs.
- Adds a regression case that asserts the pending status emitted for a reporting presubmit job in scheduling state.

## Findings

None.

## Checked

- `pkg/github/report/report.go:56-73`: the added lifecycle-state mapping uses the existing `github.StatusPending` constant and preserves the established translation boundary.
- `pkg/github/report/report_test.go:323-330`: table-driven regression coverage asserts the externally visible GitHub status for `SchedulingState`.
- Configuration, CRD/API, authorization, and tenant-routing compatibility: unchanged.
- Operational impact: status creation remains gated by normal reporting conditions; the additional update is bounded and deduplicated per state transition.

## Open questions

None for this PR.

## Followups

### [should] Fix SchedulingState handling in the Gerrit reporter

- category: deferred-review
- necessity: should
- mode: deliver
- where: `pkg/crier/reporters/gerrit/reporter.go:175-190`, `pkg/crier/reporters/gerrit/reporter.go:94-101`, `pkg/apis/prowjobs/v1/types.go:73-81`
- rationale: With `Scheduler.Enabled`, Gerrit presubmit ProwJobs start in `SchedulingState` (`pkg/gerrit/adapter/adapter.go:778`), but `ShouldReport` only skips `TriggeredState`/`PendingState`/`AbortedState`, so a job that has not run yet is reported immediately, renders through the `prohibited` fallback icon, and votes `-1` on Code-Review. `GetAllProwJobStates()` also omits `SchedulingState`, so a `SCHEDULING` line can never be parsed back by `deserialize` (reporter.go:590). This supersedes the earlier "Include SchedulingState in GetAllProwJobStates" followup.

```
In kubernetes-sigs/prow, following merged PR #951 ("Map scheduling ProwJob state to GitHub
pending status", merge commit cd1c1dbd246180e7183af5eae6cc68c2053ffcdc), fix SchedulingState
handling in the Gerrit reporter and the shared ProwJob state list.

Context:
- pkg/crier/reporters/gerrit/reporter.go: ShouldReport skips TriggeredState, PendingState and
  AbortedState but not SchedulingState. With Scheduler.Enabled, the Gerrit adapter creates
  presubmit ProwJobs in SchedulingState (pkg/gerrit/adapter/adapter.go, the
  pjutil.RequireScheduling(schedulerEnabled) call). The reporter therefore posts a review for a
  job that has not started, statusIcon falls back to the prohibited icon, and the vote logic
  selects -1 for a presubmit because report.Success != report.Total.
- pkg/apis/prowjobs/v1/types.go: GetAllProwJobStates() claims to return all possible states but
  omits SchedulingState. The Gerrit reporter's deserialize validates parsed report lines
  against it, so a scheduling line can never be read back.

Do:
1. In pkg/crier/reporters/gerrit/reporter.go, make ShouldReport return false for
   v1.SchedulingState, grouped with the existing not-done-yet TriggeredState/PendingState check.
2. Add v1.SchedulingState mapping to hourglass in the stateIcon map, so aggregated reports that
   still contain a scheduling job render it as pending rather than prohibited.
3. Add SchedulingState to GetAllProwJobStates().
4. Add focused tests: ShouldReport returns false for a scheduling Gerrit presubmit while still
   returning true for a completed one, and a report line serialized for SchedulingState
   round-trips through deserialize without error.

Acceptance criteria:
- go test ./pkg/crier/reporters/gerrit/... and go test ./pkg/apis/prowjobs/... pass.
- A scheduling Gerrit ProwJob no longer produces a Gerrit review or vote.
- The GitHub reporter mapping in pkg/github/report is unchanged.

Out of scope: the GitHub reporter, voting policy for any other state, and reporter refactors.
```

### [should] Treat SchedulingState as pending in Tide

- category: bug
- necessity: should
- mode: deliver
- where: `pkg/tide/tide.go:837-844` (`toSimpleState`), callers at `pkg/tide/tide.go:994`, `pkg/tide/tide.go:1107`, `pkg/tide/github.go:692`
- rationale: Trigger-created presubmits and Tide batches start in `SchedulingState` when the scheduler is enabled (`pkg/plugins/trigger/trigger.go:356`, `pkg/tide/tide.go:1514`), but `toSimpleState` maps scheduling to `failureState`. A queued presubmit is bucketed as missing, a queued batch is not returned as `pendingBatch`, and `getProwJobsForPRs` does not count it as pending. Now that this PR publishes a `pending` GitHub status for scheduling jobs, `isRetestEligible` can treat a Tide-created scheduling job as retest-eligible and trigger a duplicate while it waits for the scheduler.

```
In kubernetes-sigs/prow, following merged PR #951 ("Map scheduling ProwJob state to GitHub
pending status", merge commit cd1c1dbd246180e7183af5eae6cc68c2053ffcdc), fix Tide's
treatment of SchedulingState.

Context: toSimpleState in pkg/tide/tide.go maps only TriggeredState and PendingState to
pendingState, so SchedulingState falls through to failureState. With the scheduler enabled,
trigger-created presubmits and Tide batches are created in SchedulingState
(pkg/plugins/trigger/trigger.go uses pjutil.RequireScheduling(c.Config.Scheduler.Enabled);
pkg/tide/tide.go trigger() does the same). Consequently accumulate() puts such a PR in
missings/missingTests, accumulateBatch() ignores an in-flight batch, and getProwJobsForPRs()
does not count the job as a pending presubmit. PR #951 now publishes a pending GitHub status
for these jobs, which makes isRetestEligible() consider a Tide-created scheduling job as
retest-eligible and can let Tide trigger duplicates while the job is still queued.

Do:
1. In pkg/tide/tide.go, return pendingState from toSimpleState for prowapi.SchedulingState
   alongside TriggeredState and PendingState.
2. Add table-driven tests covering toSimpleState for every state, plus a test that a PR whose
   presubmit is in SchedulingState is bucketed as pending (not missing) and that a batch with a
   SchedulingState batch job is returned as pendingBatch.

Acceptance criteria:
- go test ./pkg/tide/... passes.
- A scheduling presubmit is treated as pending in accumulate() and getProwJobsForPRs(), and a
  scheduling batch is treated as pending in accumulateBatch().
- Mapping for all terminal and running states is unchanged.

Out of scope: changing when Tide creates scheduling ProwJobs, reporter changes, and retest
policy changes.
```

### [could] Allow Deck to abort ProwJobs in SchedulingState

- category: usability
- necessity: could
- mode: deliver
- where: `cmd/deck/abort.go:55-56`
- rationale: The abort allowlist accepts only `TriggeredState`/`PendingState`, so a job waiting on the scheduler cannot be aborted from Deck. Aborting is safe: the scheduler only reconciles scheduling ProwJobs and Plank does not create a pod for a ProwJob whose state is not `TriggeredState` (`pkg/plank/reconciler.go:995`).

```
In kubernetes-sigs/prow, following merged PR #951 ("Map scheduling ProwJob state to GitHub
pending status", merge commit cd1c1dbd246180e7183af5eae6cc68c2053ffcdc), allow Deck to abort
ProwJobs in SchedulingState.

Context: cmd/deck/abort.go rejects any state other than TriggeredState and PendingState, so a
job waiting for the scheduler returns "Cannot abort job with state: scheduling". Aborting such
a job is safe when the scheduler is enabled: the scheduler only reconciles scheduling
ProwJobs, and Plank does not create a pod for a ProwJob whose state is not triggered
(pkg/plank/reconciler.go).

Do:
1. In cmd/deck/abort.go, accept prowapi.SchedulingState in the abort allowlist next to
   TriggeredState and PendingState. Keep the existing error for genuinely non-abortable states.
2. Add a test case to cmd/deck/abort_test.go asserting a scheduling ProwJob is aborted (state
   becomes Aborted with the expected description), and keep or add a case proving a completed
   job is still rejected.

Acceptance criteria:
- go test ./cmd/deck/... passes.
- A scheduling ProwJob is aborted through the same flow as triggered and pending ProwJobs.
- Permission checks and the requested state transition are unchanged.

Out of scope: abort semantics for completed jobs, UI changes, and scheduler or Plank changes.
```
