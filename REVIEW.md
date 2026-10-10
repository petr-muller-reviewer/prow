---
pr: kubernetes-sigs/prow#817
title: "plank: enforce pending timeout when build cluster is unreachable"
head_sha: e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee
base: main
reviewed_at: 2026-10-05T21:46:43Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-10-10T16:43:09Z
  gated_head_sha: e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee
  reviewed_head_sha: e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee
refresh_log:
  - at: 2026-10-05T21:46:43Z
    old_sha: e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee
    new_sha: e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee
    summary: Recorded @petr-muller's non-substantive “bump (myself)” issue comment; no code or review changes.
---

# Review

## Gate

**Decision: do-not-merge.** PR #817 is open and remains at `e9b0c93495e6f8f5dcc81f4ae2716ad20b2b5eee`, the reviewed head. There are no code changes since review or activity since the previous gate. The blocking timeout regression, cache-wait error case, and missing test assertions remain in the current code.

### Findings disposition

- **Blocks merge — REVIEW.md, “Measure creation failures instead of total job age”; @petr-muller’s transient-failure/grace-period concern (`pkg/plank/reconciler.go:468-475, 753`):** The code still compares the ProwJob’s original `PendingTime` with `PodPendingTimeout`, then completes the job after one non-4xx `startPod` error. That timestamp is set on the initial transition to pending, so a transient replacement-pod `Create` failure can terminate an otherwise long-running job. Track failure duration or require repeated failures, with a regression test, before merging.
- **Hold until fixed — REVIEW.md, “Keep cache-wait failures separate from creation failures” (`pkg/plank/reconciler.go:467-475, 887-905`):** `startPod` returns errors both when `Create` fails and when a successful `Create` is followed by a cache-wait failure. The caller applies the timeout to both cases. Distinguish post-create errors before completing the ProwJob.
- **Hold until tested — REVIEW.md, “Assert the expected state in the error test case” (`pkg/plank/controller_test.go:1814-1823`):** The expected-error branch returns before asserting ProwJob state or pod count. Keep those assertions on the error path.
- **Non-gating feedback — @smg247:** `maxPodPendingTimeout` addresses the duplicated timeout lookup (`reconciler.go:859-865`). The separate suggestion to centralize error-state updates remains a maintainability cleanup, not an independent merge blocker.

### Activity since previous gate

- No commits, reviews, inline comments, or issue comments were added after the previous gate. The existing submitted reviews are `COMMENTED`; none is a `CHANGES_REQUESTED` review.

### Independent merge risk

- **Behavioral change:** The PR adds terminal-error behavior for every Plank deployment using the default 10-minute `PodPendingTimeout`. An old pending job with a missing pod can be terminated by a single transient non-4xx pod-start error. This is not opt-in, and the two-file PR diff contains no release note. This overlaps the blocking finding.
- **API and configuration:** The PR changes no exported API, Kubernetes schema, flag, or configuration default.

### Gating list

- `pkg/plank/reconciler.go:468-475, 753` — one transient pod-start error can terminate an old job based on total job age (REVIEW.md; @petr-muller).
- `pkg/plank/reconciler.go:467-475, 887-905` — a cache-wait error after successful creation can be treated as failed creation (REVIEW.md).
- `pkg/plank/controller_test.go:1814-1823` — the expected-error test skips state and pod-count assertions (REVIEW.md).
## Verdict

Request changes. The new timeout can complete a long-running job after one transient pod recreation error, or even after pod creation succeeded but its cache update was slow. The new error-path test skips its state assertions.

## What this PR does

- Complete a pending ProwJob as errored when starting a missing pod returns a non-4xx error and the job's `PendingTime` exceeds `PodPendingTimeout`.
- Otherwise, continue returning the creation error for a retry.
- Extract `maxPodPendingTimeout` and add two tests with injected pod creation errors.

**Since previous review:**

- @petr-muller posted “bump (myself)” on the issue at `2026-10-05T21:43:55Z`; there were no code changes or new review feedback.

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
