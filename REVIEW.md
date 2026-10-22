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
