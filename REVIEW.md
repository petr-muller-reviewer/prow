---
pr: kubernetes-sigs/prow#817
title: "plank: enforce pending timeout when build cluster is unreachable"
head_sha: e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee
base: main
reviewed_at: 2026-09-27T15:34:41Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-09-27T15:35:00Z
  gated_head_sha: e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee
  reviewed_head_sha: e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee
---

# Review

## Gate

**Decision: do-not-merge.** The reviewed PR head is still current, and its job-age timeout can turn one transient pod recreation failure into a terminal job error. That regression affects existing Plank deployments using the default timeout. The cache-wait case is a second way to error a job despite a successfully created pod.

### Gating findings

- **Blocks merge — REVIEW.md, `pkg/plank/reconciler.go:468-473`:** The timeout still compares `PendingTime`, which records the transition into pending, with `PodPendingTimeout`. A single replacement `Create` failure on an old job completes it. Measure failure duration or otherwise preserve retries; add a regression test.
- **Hold until fixed — REVIEW.md, `pkg/plank/reconciler.go:467-473`:** `startPod` can fail while waiting for the cache after `Create` succeeded; the same branch completes the job while its pod may run. Distinguish post-create errors from failed creations.
- **Hold until tested — REVIEW.md, `pkg/plank/controller_test.go:1814-1823`:** The expected-error test exits before its state and pod-count assertions. Assert those outcomes on the error path.

Earlier GitHub feedback is addressed: `smg247`'s duplicated-timeout concern has a helper at `reconciler.go:859-865`; the status-update helper and its unclear comment, raised by `smg247` and `petr-muller`, were removed; `petr-muller`'s cache-backed `Get` concern was addressed by moving the check to the live `Create` path. No new commits, comments, or reviews followed the saved review.

### Independent merge risk

- **Behavior:** Every Plank deployment using the default 10-minute `PodPendingTimeout` gets the new terminal-error behavior for old pending jobs whose pod must be recreated. The change is neither opt-in nor release-noted in the PR diff; one transient `Create` error can terminate a running workload. This reinforces the blocking finding.
- **API and configuration:** No exported API, CRD/schema, flag, or configuration default changed.

## Verdict

Request changes. The new timeout can complete a long-running job after one transient pod recreation error, or even after pod creation succeeded but its cache update was slow. The new error-path test skips its state assertions.

## What this PR does

- Complete a pending ProwJob as errored when starting a missing pod returns a non-4xx error and the job's `PendingTime` exceeds `PodPendingTimeout`.
- Otherwise, continue returning the creation error for a retry.
- Extract `maxPodPendingTimeout` and add two tests with injected pod creation errors.

## Findings

### [blocking] Measure creation failures instead of total job age
- where: `pkg/plank/reconciler.go:468-473`
- concern: `PendingTime` is set once when the job moves from triggered to pending (`reconciler.go:753`); it is not the start of a creation failure. If a long-running job loses its pod and one replacement `Create` attempt hits a transient non-4xx error, this branch completes the job immediately once its age exceeds `PodPendingTimeout` (10 minutes by default). The old path returned the error and retried.
- excerpt: |
    if pj.Status.PendingTime != nil && r.clock.Since(pj.Status.PendingTime.Time) >= r.maxPodPendingTimeout(pj) {
        pj.SetComplete()
        pj.Status.State = prowv1.ErrorState
- fix: Track time since pod creation first failed, or require consecutive failures, and test a long-running job whose first replacement `Create` fails.

### [should-fix] Keep cache-wait failures separate from creation failures
- where: `pkg/plank/reconciler.go:467-473`
- concern: `startPod` can return an error after `client.Create` succeeds if the pod takes more than 10 seconds to appear in the informer cache (`reconciler.go:886-904`). This new branch treats that error as a failed creation and completes an older ProwJob while its pod may already be running. The completed job no longer reconciles that pod; this branch does not delete it either.
- excerpt: |
    if !isRequestError(err) {
        if pj.Status.PendingTime != nil && r.clock.Since(pj.Status.PendingTime.Time) >= r.maxPodPendingTimeout(pj) {
            pj.SetComplete()
- fix: Apply the timeout only to failures before `Create` succeeds, and cover a successful `Create` followed by a cache-wait timeout.

### [should-fix] Assert the expected state in the error test case
- where: `pkg/plank/controller_test.go:1814-1823`
- concern: The new `ExpectError` case returns before checking `ExpectedState` or `ExpectedNumPods`. It passes for any error, including one from another path, so it does not establish that a fresh job remains pending with no pod.
- excerpt: |
    if (err != nil) != tc.ExpectError {
        if tc.ExpectError {
            t.Fatalf("expected an error from syncPendingJob, but got none")
        } else {
            t.Fatalf("syncPendingJob failed: %v", err)
        }
    }
    if err != nil {
        return
    }
- fix: Assert the ProwJob state and pod count for the expected-error case before returning.

## Checked

- Compared PR head `e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee` with merge base `7f8580d8da573bcb9e4fe716358d4c0ee1d24b38` against `upstream/main`; only `pkg/plank/reconciler.go` and `pkg/plank/controller_test.go` differ.
- `isRequestError` classifies connection failures as non-4xx, so the new branch is reachable when `client.Create` fails.
- The 4xx creation-error branch retains its previous completion behavior.
- `go test ./pkg/plank -run '^TestSyncPendingJob$' -count=1` passed.
- `git diff --check` passed for the PR diff.

## Open questions

- Can the timeout be based on the first failed creation attempt so a single failure during pod recreation cannot complete an old job?
- Can `startPod` distinguish a failed `Create` from a successful `Create` followed by a cache-wait timeout?
