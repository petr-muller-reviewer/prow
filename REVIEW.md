---
pr: kubernetes-sigs/prow#817
title: "plank: enforce pending timeout when build cluster is unreachable"
head_sha: e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee
base: main
reviewed_at: 2026-08-08T22:47:35Z
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
  - from: 3ac48b908bf172b76aba7c48afabf5b71a6bfe03
    to: e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee
    summary: >
      Force-push (old head not an ancestor of new head), single rewritten
      commit authored 2026-08-05. The mechanism changed: the timeout check
      was REMOVED from the r.pod() Get-failure branch and MOVED into
      startPod's non-request-error path inside the !podExists branch
      (reconciler.go:467-476). This resolves the previous primary blocking
      finding — Create is a live write, not a cache read, so the branch is
      genuinely reachable — and covers one of the three previously-traced
      stuck-forever paths. updateProwJobStatus was inlined back into
      syncPendingJob (helper deleted, cache-barrier comment restored);
      maxPodPendingTimeout survives at :859. The ExpectError assertion
      mismatch was fixed but an early return still skips state assertions.
      The job-age-vs-failure-duration blocking finding SURVIVES and is now
      worse: at the new call site it removes retry from the pod-recreate
      path for any job older than PodPendingTimeout, a regression against
      current behaviour. Two new findings: pod orphaning via the startPod
      cache-wait, and syncTriggeredJob left uncovered. No new PR comments
      or reviews since 2026-08-03 (only a /retest from Prucek 2026-08-05).
---

## Summary

Fixes: when pod creation against a build cluster fails with a non-4xx error (network failure, unreachable apiserver), `syncPendingJob` returns the error for requeue forever. `isRequestError` (`reconciler.go:1138-1144`) only matches 4xx `APIStatus` errors and defaults to code 500 for anything else, so a connection error never takes the "Unprocessable pod" completion path. ProwJobs stay `pending` indefinitely under controller-runtime backoff, and with `max_concurrency` set this also blocks subsequent runs.

Fix adds a timeout check inside the `!isRequestError(err)` arm of the `startPod` failure handling in the `!podExists` branch (`reconciler.go:467-476`): if `pj.Status.PendingTime` is set and `r.clock.Since(pj.Status.PendingTime.Time) >= r.maxPodPendingTimeout(pj)`, mark the job `ErrorState`/complete and fall through to the shared completion tail; otherwise return the error as before.

Verdict remains `request-changes`, but the reasons have changed substantially. The relocation is a real improvement: the branch is now reachable (the old one was not, see Resolved), and it targets a genuine stuck path. The remaining blocker is the timeout basis, which at this new call site is not merely imprecise but a behavioural regression — it deletes retry from the pod-recreate path for essentially every long-running job.

Since previous review:
- Force-push to `e9b0c9349`; `3ac48b908` is not an ancestor. `reconciler.go` +58/-71, `controller_test.go` +56/-57.
- Check moved from the `r.pod()` cache-read failure to `startPod`'s `Create` failure — resolves the "unreachable branch" blocker.
- `updateProwJobStatus` helper removed (inlined); `maxPodPendingTimeout` retained.
- No new comments or reviews; only `/retest` from Prucek on 2026-08-05T14:56:47Z.

## Findings

### [blocking] timeout gated on job age, not on how long pod creation has been failing
- where: `pkg/plank/reconciler.go:468`
- concern: `pj.Status.PendingTime` is set exactly once, in `syncTriggeredJob` at `reconciler.go:750-753`, on the `Triggered→Pending` transition, and is never reset (verified: the only two references to `PendingTime` in the file are `:468` and `:753`). It is the job's start time. Prow has no `Running` state distinct from `PendingState` (`pkg/apis/prowjobs/v1/types.go:61-62`), so `Since(PendingTime)` is total job age, not time-spent-failing-to-create-a-pod. For any job that has been alive longer than `PodPendingTimeout` — 10 minutes by default (`pkg/config/config.go:2513`) — the condition is unconditionally true.
- consequence: this is a regression, not just an imprecision. The `!podExists` branch exists for the case where a pod was deleted manually or by a rescheduler mid-flight. Today a transient non-4xx failure on the recreate — an apiserver 500, a webhook timeout, or a `getBuildID`/tot failure (`reconciler.go:867`, also a non-request error) — returns an error and retries under controller-runtime backoff. After this change, for any job past the pending timeout, that single failure completes the job as `ErrorState` with zero retries. A 3-hour job's pod gets rescheduled, one blip on the recreate, job dead.
- excerpt: |
    if !isRequestError(err) {
        if pj.Status.PendingTime != nil && r.clock.Since(pj.Status.PendingTime.Time) >= r.maxPodPendingTimeout(pj) {
            pj.SetComplete()
            pj.Status.State = prowv1.ErrorState
- fix: gate on a duration that measures the failure, not the job. Options: record a "pod creation first failed at" timestamp on the ProwJob status (set on failure, cleared on success) and compare against that; or require N consecutive failures. If neither is palatable, at minimum do not reuse `pod_pending_timeout`, whose name and existing semantics at `reconciler.go:559` both mean "the pod never started". Add a regression test: long-running job (`PendingTime` 3h ago) + single `Create` error must NOT error the job.

### [should-fix] job can be errored while its pod is actually running, and the pod is not deleted
- where: `pkg/plank/reconciler.go:467-476`, `pkg/plank/reconciler.go:893-905`
- concern: `startPod` returns a non-request error *after* a successful `Create` when the post-create cache barrier times out — `failed waiting for new pod %s in cluster %s  appear in cache` fires if the informer lags more than 10s. Previously that error was returned, and the next reconcile found the pod via `podExists` and continued normally. Now, for any job past the pending timeout, the ProwJob is completed as `ErrorState` while the pod is live in the build cluster.
- consequence: the pod runs to completion with no reporting, and unlike every other completion path in this function (`PodRunning` timeout calls `deletePod` at `:622`) nothing deletes it. Sinker eventually GCs pods whose ProwJob is complete, so it is not a permanent leak, but the job result is wrong and the cluster does the work anyway.
- excerpt: |
    }); err != nil {
        return "", "", fmt.Errorf("failed waiting for new pod %s in cluster %s  appear in cache: %w", podName.String(), pj.ClusterAlias(), err)
    }
- fix: distinguish "Create failed" from "Create succeeded, cache barrier timed out" in `startPod` (e.g. a sentinel or typed error) and only apply the timeout completion to the former; or call `deletePod` before completing.

### [should-fix] syncTriggeredJob has the identical failure path and was left unchanged
- where: `pkg/plank/reconciler.go:737-741`
- concern: `syncTriggeredJob` calls `startPod` and handles failure with the same `if !isRequestError(err) { return nil, ... }` shape, untouched by this PR. A job that has never reached `PendingState` has `PendingTime == nil`, so no timeout could apply there anyway — an unreachable build cluster still wedges triggered jobs indefinitely.
- consequence: this is arguably the *more* common manifestation of the reported symptom: a job submitted while the cluster is down never leaves `TriggeredState`. The PR only rescues jobs that were already pending when the cluster went away.
- excerpt: |
    id, pn, err = r.startPod(ctx, pj)
    if err != nil {
        if !isRequestError(err) {
            return nil, fmt.Errorf("error starting pod: %w", err)
        }
- fix: either extend the fix (needs a clock reference other than `PendingTime` — see the blocking finding, which the same mechanism would supply) or state explicitly in the PR description that triggered jobs are out of scope and why.

### [should-fix] uncovered stuck-forever paths remain from the previous review
- where: `pkg/plank/reconciler.go:604-615`, `pkg/plank/reconciler.go:820-822`
- concern: the previous pass traced three paths that wedge when a build cluster stops responding mid-flight. This push covers one (pod absent from cache → `Create` fails). Two remain:
  - **Pending, pod last seen `Running`:** `Since(pod.Status.StartTime) < maxPodRunning` (48h default, `config.go:2517`) → `return nil, nil`. No state change, no requeue, no error, no log — the job goes silent. This is the closest match to the PR body's 5-day `periodic-build-farm-canary-build06` example.
  - **Aborted:** `syncAbortedJob`'s `deletePod` fails and the error returns before `SetComplete`, so it requeues forever.
  - Related: once `PodRunningTimeout` does elapse, `deletePod` at `:622` fails and returns before the patch, so the computed `AbortedState` transition is discarded on every attempt.
- excerpt: |
    if pod.Status.StartTime.IsZero() || time.Since(pod.Status.StartTime.Time) < maxPodRunning {
        // Pod is still running. Do nothing.
        return nil, nil
    }
- fix: out of scope for this PR if deliberate, but the PR body claims to fix the canary incident and the covered path may not be the one that produced it. See the first open question — the logs settle which.

### [should-fix] ExpectError test case still asserts almost nothing
- where: `pkg/plank/controller_test.go:1814-1823`
- concern: the assertion mismatch from the previous review was fixed — `(err != nil) != tc.ExpectError` now correctly fails when an expected error is missing — but the immediately following `if err != nil { return }` skips every subsequent assertion. For the "build cluster unreachable, pending timeout not yet exceeded" case, the declared `ExpectedState: prowapi.PendingState` and `ExpectedNumPods: 0` are dead configuration. Only "some error occurred" is verified; an error from any unrelated path passes.
- excerpt: |
    if (err != nil) != tc.ExpectError {
        ...
    }
    if err != nil {
        return
    }
- fix: for the error case, still assert `tc.PJ.Status.State` and `tc.PJ.Complete()` (the in-memory object, since no patch happened) before returning, or drop the unused expectations so the test does not imply coverage it lacks.

### [nit] TerminalError from an unknown cluster alias is now absorbed by the timeout branch
- where: `pkg/plank/reconciler.go:467-476` vs `pkg/plank/reconciler.go:885`
- concern: `startPod` returns `TerminalError(fmt.Errorf("unknown cluster alias %q", ...))` at `:885`. That is not an `APIStatus` error, so `isRequestError` is false and it now lands in the new branch. If the timeout is already exceeded, the job is completed with `Pod pending timeout: failed to create pod in build cluster: ...` instead of reaching `defaultReconcile`'s terminal-error handling (`:374-388`) and its clearer `Reconciliation failed with terminal error` log. End state is the same; only the diagnostic degrades.
- fix: an `IsTerminalError(err)` guard before the timeout check, returning the error unchanged.

## Resolved

### [blocking] new branch is likely unreachable in production
- where (was): `pkg/plank/reconciler.go:499-511` (branch at `3ac48b908`)
- resolution: resolved by relocation, not by argument. The check no longer sits behind `r.pod()`, which reads from the controller-runtime informer cache and therefore does not produce connection errors. It now sits behind `startPod`'s `client.Create` (`reconciler.go:886`), a live write to the build cluster apiserver that genuinely fails with `dial tcp: connect: connection refused` when the cluster is unreachable. The previous pass's analysis of the startup probe (`pkg/flagutil/kubernetes_cluster_clients.go:394-417` excluding unreachable clusters from `r.buildClients`) still holds and still means an unreachable-*at-startup* cluster yields `TerminalError("unknown cluster alias")` — but that is a different, already-handled case. Fixed as of `e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee`.

### [should-fix] new branch bypasses shared completion tail
- where (was): `pkg/plank/reconciler.go:463-476` at `6e36518`
- resolution: resolved structurally. The new branch sets state and falls through to the same inline tail (JobURL at `:640`, transition log, patch, cache barrier) as every other completion path in `syncPendingJob`, so there is nothing to bypass. The `updateProwJobStatus` helper introduced at `3ac48b908` was deleted in this push. Fixed as of `e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee`.

### [should-fix] updateProwJobStatus name does not describe what it does
- resolution: moot — the helper no longer exists.

### [nit] cache-barrier rationale comment dropped during extraction
- resolution: the comment is back inline at `reconciler.go:656-658`, adjacent to the barrier it explains. Fixed as of `e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee`.

### [should-fix] duplicated maxPodPending computation
- resolution: `r.maxPodPendingTimeout(pj)` retained at `reconciler.go:859-864`; used at `:468` and `:559`. Still fixed.

### [nit] err variable reused for two different meanings
- resolution: the shadowing branch no longer exists. The `err` at `:466` is `startPod`'s own, scoped to the `!podExists` block.

## Checked

- `isRequestError` (`reconciler.go:1138-1144`) defaults to code 500 for non-`APIStatus` errors and returns false — confirming a connection error takes the new branch, not the "Unprocessable pod" branch. The PR description's account of the bug is accurate.
- The `else` restructuring at `:477-482` preserves the pre-existing 4xx behaviour byte-for-byte (`ErrorState`, `SetComplete`, `Pod can not be created: %v`, `Unprocessable pod.` warning) — only the nesting changed.
- Falling through from the timeout branch is safe: the tail at `:633-637` guards `pod != nil` before dereferencing, and `pj.Complete()` is already true so it does not double-set.
- New code uses `r.clock.Since` (fake-clock-aware) where the surrounding `PodRunning` check at `:612` still uses bare `time.Since`.
- `maxPodPendingTimeout` correctly prefers `pj.Spec.DecorationConfig.PodPendingTimeout` over `Plank.PodPendingTimeout`; both are non-nil after defaulting.
- Timeout defaults confirmed: `PodPendingTimeout` 10m, `PodRunningTimeout` 48h, `PodUnscheduledTimeout` 5m (`pkg/config/config.go:2513-2521`).
- Test harness now drives the new branch via `clientWrapper.createError` (`controller_test.go:1795-1797`); the `getError` field and the `GetErr` test-case field were removed. `pjClient` is `fakeMgr.GetClient()`, so the ProwJob patch path is unaffected by the injected error.
- The two new test cases give the PJ a real `PodSpec` and `Refs` (unlike the `3ac48b908` versions, which had an empty `Spec`), so `decorate.ProwJobToPod` is actually exercised before `Create` fails.
- `ExpectedURL` is not asserted by `TestSyncPendingJob`'s body, so omitting it on the new cases is harmless.
- No config-schema change; rollback is a plain revert, no persisted-state format change.
- No security surface: no new inputs, credentials, or external calls.
- NOT verified this pass: `gofmt`, `go build`, `go vet`, `go test ./pkg/plank/...` — no Go toolchain in this environment. Treat build/test status as unknown.

## Open questions

- Still highest priority, still unanswered from the previous pass: what do plank's logs actually say for the stuck canary (`periodic-build-farm-canary-build06`, the 5-day example in the PR body)? If they show `error starting pod for PJ ...`, this PR now targets the right path and the review reduces to the findings above. If they show silence or `Reconciliation failed with terminal error`, the covered path is not the incident's path. Nothing else can be settled without this.
- Was the job-age reading of `PendingTime` deliberate? It cannot distinguish "pod never started" from "job has been running for hours", and at the new call site the latter is the common case.
- Is losing retry on the pod-recreate path acceptable? Before this change a transient `Create` failure retried under backoff; after it, for any job past the pending timeout, the first failure is fatal. If not intended, the blocking finding's fix (measure failure duration, or require N consecutive failures) also restores it.
- Should triggered-but-not-yet-pending jobs be covered too, or is the `TriggeredState` wedge deliberately left for a follow-up?
- Worth a distinct metric or log for "errored because pod creation failed against an unreachable cluster", to separate it from other `ErrorState` causes? It can fire across many jobs at once.
- Should mid-flight unreachability get explicit handling at all — re-probing clusters that were healthy at startup, extending the existing `interrupts.Terminate()` recovery in `kubernetes_cluster_clients.go:427-462` — rather than relying on each job's own timeout?
