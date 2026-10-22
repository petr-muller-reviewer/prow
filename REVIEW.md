---
pr: kubernetes-sigs/prow#878
title: "test/integration: deflake TestDeck against parallel deck restarts"
head_sha: 43e5937599c01c879013cba2064edff47282ffce
base: main
reviewed_at: 2026-08-27T10:31:27Z
verdict: request-changes
---

## What this PR does

- Fixes a real intermittent CI flake: `TestDeck` raced `TestRerun`'s parallel deck-deployment restart and lost when the ingress was still returning 5xx.
- Makes `TestDeck`'s front-page fetch poll for up to 180s (mirroring `TestDeckTenantIDs`) instead of failing on the first non-200/error.
- Fixes a nil-pointer deref in `TestRerun`'s retry helper: a transport-level error left `res` nil, and `res.Body.Close()` panicked instead of retrying.
- Makes `refreshProwPods` wait (poll up to 2 min) for the old pods to be gone and a replacement to report Ready before returning, instead of returning immediately after issuing deletes.
- Lowers deck's `readinessProbe.timeoutSeconds` from 600 to 3 (kubelet runs probes serially per container; a single hung probe could freeze the Ready condition for 10 minutes).

## Findings

### [should-fix] TestDeck's poll doesn't bind the underlying HTTP call to the context/deadline
- where: `test/integration/test/deck_test.go:227-243`
- concern: The scraper passed to `wait.PollUntilContextTimeout` calls `http.Get(url)` (no context), instead of `http.NewRequestWithContext(ctx, ...)` + `http.DefaultClient.Do`. `PollUntilContextTimeout` only checks the deadline between condition-function calls, not while one is in flight. If the ingress/deck ever black-holes the connection (TCP-level hang, as opposed to a quick connection-refused or 5xx), `http.Get` can block indefinitely and the intended 180s ceiling is not actually enforced — the test hangs until the outer `go test` binary timeout kills the whole suite rather than failing this test cleanly.
- excerpt: |
    scraper := func(ctx context.Context) (bool, error) {
        resp, err := http.Get("http://localhost/deck")
        ...
    }
    if waitErr := wait.PollUntilContextTimeout(context.Background(), 1*time.Second, 180*time.Second, true, scraper); waitErr != nil {

### [should-fix] refreshProwPods now holds prowComponentsMux for the entire readiness wait, serializing independent parallel restarts
- where: `test/integration/test/setup.go:132-175`
- concern: The PR description acknowledges the lock is now held longer ("~10-15s for deck") and calls it absorbed by callers' minute-scale budgets, but the wait is bounded at up to 2 minutes, not 10-15s in the worst case. `TestLaunchProwJob` and `TestRerun` both call `refreshProwPods` and both run via `t.Parallel()`; with the wider critical section, one test's restart-and-wait can now fully serialize behind the other's for up to 2 minutes in the unhappy path, which is a meaningfully different (and untested) failure mode from the previous brief list+delete lock hold.
- excerpt: |
    prowComponentsMux.Lock()
    defer prowComponentsMux.Unlock()
    ...
    return wait.PollUntilContextTimeout(ctx, 2*time.Second, 2*time.Minute, true, func(ctx context.Context) (bool, error) {

## Checked

- `TestRerun` nil-deref fix: `res` is genuinely nil only on transport-level errors (confirmed `http.DefaultClient.Do` contract), so skipping `res.Body.Close()` on that branch and retrying instead is correct.
- `labels.Parse` error handling fix (previously silently discarded via `_`) is a straightforward correctness improvement, no concerns.
- Readiness-probe `timeoutSeconds` change (600 → 3) matches `periodSeconds: 3` already in place; deck's `/healthz/ready` is cheap, flapping risk is low as claimed.
- The `<title>Prow Status</title>` assertion in `TestDeck` intentionally stays outside the poll loop, so a served-but-wrong page still fails immediately — correct per the stated intent.

## Open questions

- For the `refreshProwPods` lock-hold widening: was serialization between `TestLaunchProwJob` and `TestRerun` (rather than just parallelism within each) actually exercised in a flaky-CI repro, or is the "absorbed by minute-scale budgets" claim untested? Worth confirming the worst case (both tests' deck/horologium restarts landing back-to-back) still fits comfortably under the suite's 10-minute binary timeout mentioned in "known limitations".
- Any reason not to bind the `TestDeck` scraper's `http.Get` to the poll's `ctx` (via `http.NewRequestWithContext`) so a network-level hang is actually bounded by the 180s, matching the intent stated in the fatal message?

## Followups

Accepted: 3 (2 deliver, 1 probe). Skipped: 0.

PR #878 merged on 2026-08-26 at `3dab1c8551d4cd9272f7491472bcd18f470d2a6d`. The opportunities below were checked against the merged files. The first two carry forward the unresolved review findings; the third addresses adjacent retry cleanup. No implementation work was performed during this followup walk.

### Enforce Deck scraper deadlines

- Category: reliability
- Necessity: should — stalled HTTP requests can outlive the advertised 180-second poll deadline.
- Where: `test/integration/test/deck_test.go:235` and `test/integration/test/deck_test.go:315`.
- Why followup: the merge handles transient 5xx responses, but both the new front-page scraper and the existing tenant scraper still omit the poll context from their HTTP requests.
- Decision: accepted.
- Mode: deliver

```text
In kubernetes-sigs/prow, following PR #878 — "test/integration: deflake TestDeck against parallel deck restarts" (merge commit 3dab1c8551d4cd9272f7491472bcd18f470d2a6d), work from the merged default branch. Recheck the current code before editing because later changes may have addressed this followup.

In test/integration/test/deck_test.go, bind the GET requests in TestDeck and TestDeckTenantIDs to each poll callback's context using http.NewRequestWithContext and the HTTP client. Ensure cancellation covers both waiting for headers and reading the response body. Preserve retries for transient request, status, and body-read failures; close response bodies; keep tenant assertions as they are and the front-page title assertion outside the poll.

Acceptance criteria: focused validation demonstrates cancellation during stalled headers and stalled body reads, plus a transient failure followed by success. The existing 180-second polling budgets remain effective and the current assertions retain their meaning. Account for the package TestMain's --run-integration-test gate when running focused tests so they actually execute; avoid requiring a live cluster for controlled HTTP cases where practical.

Scope guard: limit changes to these two scrapers and necessary focused coverage. Do not change global HTTP defaults, other integration tests, or suite timeouts. If already addressed on the default branch, report the existing fix instead of duplicating it.
```

### Probe restart readiness and contention

- Category: tests
- Necessity: should — resolve the saved concern about readiness waits and restart serialization with evidence.
- Where: `test/integration/test/setup.go:132`, `test/integration/test/horologium_test.go:72`, and `test/integration/test/deck_test.go:409`.
- Why followup: the merged helper holds the global component mutex throughout the readiness wait. Same-component coordination is intentional, but queued waits and their effect on caller deadlines and the suite budget remain unverified.
- Decision: accepted as probe.
- Mode: probe

```text
In kubernetes-sigs/prow, following PR #878 — "test/integration: deflake TestDeck against parallel deck restarts" (merge commit 3dab1c8551d4cd9272f7491472bcd18f470d2a6d), work from the merged default branch. Recheck the current refreshProwPods implementation and callers before starting.

Use temporary focused tests with controlled client responses to exercise refreshProwPods in test/integration/test/setup.go. Cover an old pod UID still present, no replacement, an unready replacement, a terminating replacement, and a ready replacement after all old pods disappear. Exercise overlapping horologium/horologium and horologium/deck calls, slow readiness, and cancellation while queued. Measure mutex queue time separately from the two-minute readiness poll. Inspect TestLaunchProwJob in horologium_test.go, TestRerun in deck_test.go, and the actual runner configuration to assess whether supported overlapping restart scenarios threaten the suite's ten-minute budget. Account for TestMain's --run-integration-test gate so focused tests actually execute.

Acceptance criteria: exercise every named case and classify each failure as an implementation bug or an incorrect test expectation, with evidence. Real bugs include returning ready while old pods remain or without a usable ready replacement, unsafe overlapping deletion, or a demonstrated violation of the intended cancellation or runtime contract. Serialization and queue time alone do not establish a bug; distinguish deliberate same-component protection from demonstrated failures. If a real bug is found, apply only its minimal fix and report the failing case. If none is found, leave no files changed.

Scope guard: the tests are a probe, not the deliverable. Delete all temporary probe tests and instrumentation before finishing. Preserve protection against overlapping restarts of the same component. Do not blindly release the mutex before readiness, weaken readiness checks, increase suite timeouts, fix test-only problems, or refactor unrelated code. Submit changes only for a demonstrated implementation defect; otherwise report the evidence and leave the checkout unchanged.
```

### Finish rerun retry resource handling

- Category: cleanup
- Necessity: could — close successful response bodies and remove 40 seconds of avoidable delay after the final failed attempt.
- Where: `test/integration/test/deck_test.go:461`.
- Why followup: the merge fixes the transport-error panic, while adjacent pre-existing response cleanup and backoff behavior remain in the same loop.
- Decision: accepted.
- Mode: deliver

```text
In kubernetes-sigs/prow, following PR #878 — "test/integration: deflake TestDeck against parallel deck restarts" (merge commit 3dab1c8551d4cd9272f7491472bcd18f470d2a6d), work from the merged default branch. Recheck the current code before editing because later changes may have addressed this followup.

Update TestRerun's local retry helper in test/integration/test/deck_test.go so every received response body is closed, including after a successful read, and backoff occurs only when another attempt remains. Preserve three attempts, the current delays between attempts, retries for transport errors and non-200 responses, and the existing treatment of body-read failures. The current loop sleeps for 10, 20, and 40 seconds on three consecutive failures; retain the first two delays and eliminate the final sleep.

Acceptance criteria: focused validation demonstrates successful-body closure, correct cleanup on failure paths, and exhausted retries without a final sleep. Transport errors with a nil response must remain safe. Account for TestMain's --run-integration-test gate when running focused tests so they actually execute.

Scope guard: keep changes within the rerun retry lifecycle and necessary focused coverage. Do not redesign rerun semantics, broaden retry policy, change suite timeouts, or refactor other integration tests. If already addressed on the default branch, report the existing fix instead of duplicating it.
```
