---
pr: kubernetes-sigs/prow#817
title: "plank: enforce pending timeout when build cluster is unreachable"
head_sha: 3ac48b908bf172b76aba7c48afabf5b71a6bfe03
base: main
reviewed_at: 2026-08-03T21:37:02Z
verdict: request-changes
refresh_log:
  - from: 6e3651863113d1e8e432d295a0bc1ac88dda7d3d
    to: 3ac48b908bf172b76aba7c48afabf5b71a6bfe03
    summary: >
      Author refactored to extract r.maxPodPendingTimeout(pj) and
      r.updateProwJobStatus(ctx, pj, prevPJ) helpers, deduplicating the
      maxPodPending computation and routing the new error branch through
      the same completion tail (JobURL, transition log, patch, cache-sync
      wait) as the rest of syncPendingJob. Resolves both should-fix findings
      and the err-shadowing nit. The blocking finding (timeout gated on
      pj.Status.PendingTime, i.e. job age, not on cluster-unreachable
      duration) is NOT addressed — same logic, just relocated. Maintainer
      smg247 left review comments independently converging on the two
      duplication should-fix findings (now resolved by this push); no
      comment yet on the blocking issue.
  - from: 3ac48b908bf172b76aba7c48afabf5b71a6bfe03
    to: 3ac48b908bf172b76aba7c48afabf5b71a6bfe03
    summary: >
      No code change. Reviewer (petr-muller) posted the blocking finding as
      a PR issue comment on 2026-08-03T14:19:29Z, questioning whether a
      single transient Get failure should kill a job. Follow-up analysis
      confirmed buildClient.Get is a controller-runtime cache/informer read
      (buildCluster.GetCache() is wired via WatchesRawSource), not a live
      per-call API request, which narrows the blocking finding's real
      trigger from "any transient blip" to "a plank restart whose informer
      has not yet completed initial sync while the build cluster is down".
  - from: 3ac48b908bf172b76aba7c48afabf5b71a6bfe03
    to: 3ac48b908bf172b76aba7c48afabf5b71a6bfe03
    summary: >
      No code change. Full re-review pass. CORRECTION to the previous
      refresh: the "plank restart with unsynced informer" trigger does not
      exist. pkg/flagutil/kubernetes_cluster_clients.go:394-417 probes each
      build cluster at startup (cluster.New + live SelfSubjectAccessReview)
      and EXCLUDES unreachable ones from the returned map, so they never
      enter r.buildClients; r.pod() then returns TerminalError("no build
      client found"), which defaultReconcile already completes today. Net
      effect: no reachable trigger for the new branch has been identified,
      making it likely dead code — promoted to the primary blocking
      finding, absorbing the old trigger analysis. Traced the actual
      unreachable-mid-flight behaviour instead and identified three
      stuck-forever paths the PR does not touch. Added should-fix on the
      ExpectError test asserting nothing, should-fix on
      updateProwJobStatus naming/godoc, and a question on retranslating to
      TerminalError as a simpler implementation.
---

## Summary

Fixes: when the build cluster is unreachable, `syncPendingJob` returns the `r.pod()` error immediately, so the `pod_pending_timeout` check (which only runs after a successful pod fetch) never fires. ProwJobs stay `pending` forever under controller-runtime's exponential backoff, and with `max_concurrency` set this also blocks subsequent runs of the job.

Fix adds a timeout check in the `err != nil` branch of `syncPendingJob` (`reconciler.go:499-511`): computes `maxPodPending` via the new `r.maxPodPendingTimeout(pj)`, compares it against `r.clock.Since(pj.Status.PendingTime)`, and if exceeded marks the job `ErrorState`/complete and patches it via the new shared `r.updateProwJobStatus` instead of propagating the error.

Verdict remains `request-changes`. The refactor (both extracted helpers) is good and resolves all prior should-fix findings. The problem is now upstream of the timeout semantics: this review pass could not identify any production condition under which the modified branch executes, and traced three separate stuck-forever paths that the PR does not touch.

Since previous review:
- No new commits, comments, or reviews. Head SHA unchanged at `3ac48b9`.
- Trigger analysis corrected (see `refresh_log`): unreachable-at-startup yields `TerminalError`, not a `Get` error. Prior blocking finding on the timeout basis is retained but demoted to second position — it only matters once the branch can fire at all.
- Traced full "build cluster stops responding mid-flight" behaviour; three uncovered stuck paths identified.

## Findings

### [blocking] new branch is likely unreachable in production
- where: `pkg/plank/reconciler.go:499-511` (branch), `pkg/plank/reconciler.go:845` (the erroring call), `pkg/flagutil/kubernetes_cluster_clients.go:394-417` (startup probe)
- concern: the branch only runs when `buildClient.Get` returns a non-`NotFound` error, and no path producing one has been identified. Two sub-cases exhaust the "build cluster unreachable" space:
  - **Unreachable at plank startup:** `BuildClusters` probes each cluster (`cluster.New` plus a live `CheckAuthorizations` SSAR) and omits failures from the returned map; `cmd/prow-controller-manager/main.go:175-177` logs at `Error` and continues. The cluster never enters `r.buildClients`, so `r.pod()` returns `TerminalError("no build client found for cluster %q")` at `reconciler.go:835` and `defaultReconcile:374-388` already completes the job today. Corroborated by `syncClusterStatus` reporting this exact state as `ClusterStatusNoManager` (`reconciler.go:319-322`).
  - **Unreachable after startup:** `bc.Client = buildCluster.GetClient()` (`reconciler.go:163`) with pods watched via `GetCache()` (`reconciler.go:154-159`), controller-runtime v0.21 (`go.mod:83`), and no `DisableFor`/`CacheOptions` overrides anywhere in `cmd/` or `pkg/flagutil/`. Reads are served from the informer indexer with no per-call network I/O, so `Get` keeps succeeding against frozen data across watch loss. An unsynced informer would *block* in `WaitForCacheSync` rather than error, returning only on context cancellation at shutdown.
- consequence: the test's injected `errors.New("dial tcp: connect: connection refused")` (`controller_test.go:1417`) is a shape this call site cannot produce in production, so the green test does not demonstrate the bug is fixed.
- caveat: controller-runtime is not vendored and there is no module cache in this checkout, so the informer/`WaitForCacheSync` semantics are from library knowledge, not verified against v0.21 source. Everything cited with `file:line` is verified here.
- fix: establish the real failure path from plank logs for the PR's own example (`periodic-build-farm-canary-build06`, stuck 5 days) before merging, then fix the branch that actually fires. See the next finding for candidates.

### [blocking] three stuck-forever paths exist that the PR does not cover
- where: `pkg/plank/reconciler.go:656-658`, `pkg/plank/reconciler.go:514-521`, `pkg/plank/reconciler.go:822`
- concern: traced behaviour when a build cluster stops responding mid-flight — reads keep succeeding from the frozen cache, writes (`Create`/`Delete`) fail for real, pod events stop:
  - **Pending, pod last seen `Running`:** `Since(pod.Status.StartTime) < maxPodRunning` (default 48h, `config.go:2517`) → `return nil, nil`. No state change, no requeue, no error, no log. The job goes completely silent. This is the closest match to the PR's 5-day example.
  - **Pending, pod absent from cache:** `podExists == false` → `startPod` → `Create` is a direct write and fails; `isRequestError` (`reconciler.go:1146-1152`) sees no `APIStatus`, defaults to code 500, returns false → `error starting pod for PJ %s` → backoff requeue forever, `pod_pending_timeout` never consulted.
  - **Aborted:** `syncAbortedJob` `Delete` fails at `:822`, error returned before `SetComplete` → requeue forever, never completes.
  - Also: once `PodRunningTimeout` finally elapses on a resync-triggered reconcile, `deletePod` fails and the error returns *before* `updateProwJobStatus`, so the `AbortedState` transition is computed and then discarded on every attempt.
- excerpt: |
    case corev1.PodRunning:
        ...
        if pod.Status.StartTime.IsZero() || time.Since(pod.Status.StartTime.Time) < maxPodRunning {
            // Pod is still running. Do nothing.
            return nil, nil
        }
- note: plank does not self-restart in this regime. The recovery goroutine in `kubernetes_cluster_clients.go:427-462` only launches `if aggregatedErr != nil` and only probes clusters missing from `res`, so a cluster healthy at startup is never re-probed and `interrupts.Terminate()` never fires. Only a kubeconfig change (`main.go:193-198`) restarts plank. `syncClusterStatus` writing `ClusterStatusError` every minute is the sole operator signal.
- fix: pick the path that matches the observed incident. If it is the `Running`-pod silence, the fix belongs near `:656`; if it is pod-recreation, near `:514`.

### [blocking] timeout gated on job age, not on cluster-unreachable duration
- where: `pkg/plank/reconciler.go:501`
- concern: applies only once the branch can fire at all, but is independently wrong. Two defects relative to how `maxPodPending` is used everywhere else:
  1. **Wrong clock reference.** `pj.Status.PendingTime` is set exactly once at `Triggered→Pending` (`reconciler.go:769`) and never updated. Prow has no distinct "Running" ProwJobState — `PendingState`'s doc comment (`pkg/apis/prowjobs/v1/types.go:61-62`) reads "the job is currently running and we are waiting for it to finish." So `Since(PendingTime)` is total job age, not time-in-pod-pending and not time-since-unreachable.
  2. **Lost phase guard.** The existing check at `reconciler.go:626` compares against `pod.Status.StartTime` and only runs inside `case corev1.PodPending`. The new branch has no pod, so it applies unconditionally — including to jobs whose pod was last seen `Running`.
- consequence: a job whose pod has run 3h has `Since(PendingTime) == 3h >= 30m`, so the first observed `Get` failure marks it `ErrorState`/complete. Default is 10m (`config.go:2513`); the PR's own example uses 30m. Both are shorter than most CI job runtimes, so this would hit nearly the entire in-flight population of a cluster at once.
- excerpt: |
    if pj.Status.PendingTime != nil && r.clock.Since(pj.Status.PendingTime.Time) >= r.maxPodPendingTimeout(pj) {
- fix: gate on a duration that measures unreachability — a "first Get failure at" timestamp (set on failure, cleared on success), or N consecutive failures. Do not reuse `pod_pending_timeout`, whose name and existing semantics both mean "the pod never started". Add a test: long-running job + single `Get` error must NOT error the job.

### [should-fix] ExpectError test case asserts nothing
- where: `pkg/plank/controller_test.go:1808-1818`
- concern: the early `return` skips every subsequent assertion, so `ExpectedState: prowapi.PendingState` on the "pending timeout not yet exceeded" case is dead configuration. Nothing verifies the ProwJob was left untouched, which is the actual behaviour under test.
- excerpt: |
    reconcileResult, err := r.syncPendingJob(ctx, &tc.PJ)
    if tc.ExpectError && err != nil {
        return
    }
- fix: assert the error, then fall through to the state/pod checks, or at minimum check `tc.PJ.Status.State` and `tc.PJ.Complete()` before returning.

### [should-fix] updateProwJobStatus name does not describe what it does
- where: `pkg/plank/reconciler.go:461-492`
- concern: the name reads as a setter but the function (1) mutates `pj.Status.URL`, (2) logs the transition, (3) patches, and (4) if the state changed, blocks up to 2s polling until the cache reflects it — overwriting `pj` with the object read back. Neither the `pj` mutation nor the blocking barrier is inferable from the name, and the barrier is the whole point of the function.
- fix: preferred — hoist the `JobURL` assignment back to call sites and rename to `patchProwJobAndWaitForCache`, which then fully describes the body. Alternative — keep as-is under `finalizeProwJobStatus`/`commitProwJobStatus` plus a godoc stating that `pj` is mutated, that the URL error is logged and swallowed, and why the cache barrier exists (a later reconcile could otherwise observe pre-patch state and recreate a just-deleted pod).

### [nit] cache-barrier rationale comment dropped during extraction
- where: `pkg/plank/reconciler.go:477-489`
- concern: the explanatory comment moved out of `syncPendingJob` but did not land in `updateProwJobStatus`. It is the only place the non-obvious barrier is justified.
- excerpt: |
    // If the ProwJob state has changed, we must ensure that the update reaches the cache before
    // processing the key again. Without this we might accidentally replace intentionally deleted pods
    // or otherwise incorrectly react to stale ProwJob state.

### [question] retranslate to TerminalError instead of hand-rolling completion?
- where: `pkg/plank/reconciler.go:499-511` vs `pkg/plank/reconciler.go:374-388`
- concern: `return nil, TerminalError(fmt.Errorf("pending timeout exceeded, build cluster unreachable: %w", err))` would reuse the existing terminal handler and collapse the whole branch to ~4 lines, removing the need for `updateProwJobStatus` in this PR. Verified it propagates: `syncPendingJob` → `reconcile:326` → `serializeIfNeeded:422` → `defaultReconcile:374`, no wrapping; `IsTerminalError` works despite the `*nonRetryableError`/`nonRetryableError{}` pointer-value mismatch (value-receiver `Is` is in the pointer method set).
- tradeoffs: loses the cache-sync barrier — and unlike existing terminal errors (unknown alias, idempotent), "cluster unreachable" can stop being true, so a stale re-read could hit `podExists == false` and start a fresh pod for an already-errored job. Also swallows patch failures (`:384-386` logs and returns nil, no requeue), reintroducing the stuck-forever symptom. `Status.URL` loss is moot (already set at `:773`). Would also need `if IsTerminalError(err) { return nil, err }` first to avoid double `nonretryable error:` prefixes.

### [question] TerminalError folded into the same branch
- where: `pkg/plank/reconciler.go:499-511` vs `pkg/plank/reconciler.go:835`
- concern: if the timeout is already exceeded and the failure is a `TerminalError` (unknown cluster alias), the new branch handles it directly and reports "could not get pod from build cluster" rather than `defaultReconcile`'s clearer "Terminal error: ...". End state is identical; only the diagnostic degrades. Intentional, or worth an `IsTerminalError(err)` guard?

## Resolved

### [should-fix] duplicated maxPodPending computation
- where (was): `pkg/plank/reconciler.go:459-462` vs `pkg/plank/reconciler.go:569-572`
- resolution: extracted `func (r *reconciler) maxPodPendingTimeout(pj *prowv1.ProwJob) time.Duration`; both call sites use it. Fixed as of `3ac48b908bf172b76aba7c48afabf5b71a6bfe03`.

### [should-fix] new branch bypasses shared completion tail
- where (was): `pkg/plank/reconciler.go:463-476` vs `pkg/plank/reconciler.go:646-680`
- resolution: extracted `func (r *reconciler) updateProwJobStatus(ctx context.Context, pj, prevPJ *prowv1.ProwJob) error` doing JobURL, transition log, patch, and cache-sync wait. Both call sites use it. Fixed as of `3ac48b908bf172b76aba7c48afabf5b71a6bfe03`. (Naming follow-up above.)

### [nit] err variable reused for two different meanings
- where (was): `pkg/plank/reconciler.go:466-471`
- resolution: `updateProwJobStatus` declares its own local `err`, so the pod-fetch `err` is no longer shadowed. Fixed as of `3ac48b908bf172b76aba7c48afabf5b71a6bfe03`.

## Checked

- `r.pod()` returns `(nil, false, nil)` on `NotFound` (`reconciler.go:846-848`), so the new branch is correctly scoped to real fetch failures, not "pod doesn't exist yet".
- No `deletePod` in the new branch — correct, the cluster is unreachable. Pod is not leaked permanently: sinker GCs pods whose ProwJob is complete.
- `clientWrapper.getError` wraps the build cluster client only; `pjClient` is `fakeMgr.GetClient()` (`controller_test.go:1781`), so the ProwJob patch path is unaffected by the injected error.
- New code uses `r.clock.Since` (fake-clock-aware) where surrounding code still uses bare `time.Since` — an improvement.
- Timeout defaults confirmed: `PodPendingTimeout` 10m, `PodRunningTimeout` 48h, `PodUnscheduledTimeout` 5m (`pkg/config/config.go:2513-2521`).
- No config-schema change; `Plank.PodPendingTimeout` / `DecorationConfig.PodPendingTimeout` reused as-is, both non-nil after defaulting. Rollback is a plain revert, no persisted-state format change.
- No security surface: no new inputs, credentials, or external calls.
- Confirmed `pkg/apis/prowjobs/v1/types.go:61-62` that Prow has no "Running" ProwJobState distinct from `PendingState`.
- NOT verified this pass: `gofmt`, `go build`, `go vet`, `go test ./pkg/plank/...` — no Go toolchain in this environment. Treat build/test status as unknown.

## Open questions

- Highest priority: what do plank's logs actually say for the stuck canary (`periodic-build-farm-canary-build06`, the 5-day example in the PR body)? Expect one of `error starting pod for PJ ...` (fix belongs in the `startPod` path), `failed to get pod: ...` (the cache analysis above is missing something — worth understanding before merge), or `Reconciliation failed with terminal error` (the job was not stuck for the assumed reason). Nothing else in the review can be settled without this.
- Was the job-age interpretation of the timeout deliberate? With `PendingTime` never reset and no `Running` state, it cannot distinguish "pod never started" from "job has been running for hours".
- Interest in a follow-up requiring N consecutive failures (or a distinct grace-period config) before erroring, so one event does not fail every in-flight job on a cluster at once?
- Worth a distinct metric/log for "errored because build cluster unreachable", to separate it from other `ErrorState` causes once it can fire in bursts?
- Should the mid-flight-unreachable case get explicit handling at all (e.g. re-probing clusters that were healthy at startup, extending the existing `interrupts.Terminate()` recovery), rather than relying on each job's own timeout?
