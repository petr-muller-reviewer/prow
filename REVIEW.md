---
pr: kubernetes-sigs/prow#906
title: "entrypoint: de-flake interrupt tests and fix SIGKILL-path data race"
head_sha: 9a34b2ed01df1575a195a9468872ed9a596a04ac
base: main
reviewed_at: 2026-08-27T10:31:14Z
verdict: approve
---

## What this PR does

- Fixes a data race on `command.ProcessState`: after a timeout/interrupt, the code used to read `ProcessState.ExitCode()` unconditionally even when the `Wait()` goroutine hadn't necessarily finished writing it.
- Introduces an `exited` flag, set only when the code actually received from `done`, and routes exit-code propagation through a new `propagatedExitCode(command, exited)` helper that returns `-1` (matching `ExitCode()`'s own signal-killed convention) when `exited` is false, instead of racing the read.
- Makes `done` a buffered channel (`chan error, 1`) so the `Wait()` goroutine's send can never block/leak once the main goroutine stops receiving after the grace period.
- Adds a bounded second wait (`sigkillDrainTimeout`, hardcoded 1s) after `SIGKILL` in `gracefullyTerminate`, to avoid blocking forever if an orphaned descendant is still holding the log pipe open; `gracefullyTerminate` now returns a `bool` indicating whether `Wait()` actually completed (i.e., whether `ProcessState` is safe to read).
- De-flakes `run_test.go`'s interrupt tests, replacing EXIT-trap/background-job test scaffolding that was itself flake-prone.

## Findings

### [nit] `sigkillDrainTimeout` is a hardcoded constant, unlike the other timeouts
- where: `pkg/entrypoint/run.go:68`
- concern: `Timeout` and `GracePeriod` are both user-tunable via `Options` fields (defaulted via `optionOrDefault`), but `sigkillDrainTimeout` is a bare 1s constant with no override. A wrapped process whose legitimate (non-buggy) child takes slightly over 1s to release the log pipe after SIGKILL will have its real exit code silently replaced by `-1`, with no way for the job author to raise this window the way they can already raise `GracePeriod`/`Timeout`.
- excerpt: |
    // sigkillDrainTimeout bounds how long we wait for Wait to return
    // after SIGKILL. Death and reap of the killed process are
    // near-instant; if this expires, an orphaned descendant is holding
    // the log pipe open and Wait may block arbitrarily long.
    sigkillDrainTimeout = 1 * time.Second

## Checked

- `propagatedExitCode`'s race-safety: only a receive from `done` (setting `exited`) synchronizes-before the `Wait()` goroutine's write to `ProcessState`; the `-1` fallback path is correctly race-free.
- `done` channel buffering: change from unbuffered to `chan error, 1` correctly prevents the `Wait()` goroutine leaking/blocking after the main goroutine stops selecting on it.
- `gracefullyTerminate`'s new `bool` return value is threaded correctly through both call sites (timeout and interrupt paths) into `propagatedExitCode`.
- Removed EXIT-trap/background-job test behavior in `run_test.go` is pure flake-inducing test scaffolding, not a production-behavior regression; no invariant was dropped.
- No duplicated logic or hot-path cost introduced; nothing to simplify.
- No repo CLAUDE.md/convention governs this path.

## Open questions

- Would you consider making `sigkillDrainTimeout` configurable (or at least documented as a known limitation) for wrapped commands with legitimately slower log-pipe teardown?

## Followups

PR #906 merged on 2026-08-26 as `b9f26284d47bc7e7ba0b726c3b13144b4982ab29`. The reviewed implementation, tests, configuration, and user guide were checked against the merged change. No deferred human review discussion was found.

Accepted: 1 (probe). Skipped: 1 (documenting the fixed SIGKILL drain deadline). The skipped candidate came from the existing configurability nit; that finding remains open.

### Probe both SIGKILL drain outcomes with controlled pipe lifetimes

- Category: tests
- Necessity: should — verify that both post-kill drain outcomes are exercised reliably.
- Mode: probe
- Where: `pkg/entrypoint/run_test.go:340`, `pkg/entrypoint/run.go:163`, and `pkg/entrypoint/run.go:277`.
- Why followup: the retained-pipe fixture uses `sleep 3`, so scheduling and readiness delays can allow the descendant to exit before the drain deadline. The successful-drain case checks the propagated code but does not assert that the timeout diagnostic is absent. Probe both cases with controlled fixtures to investigate the production guarantees.

```text
Mode: probe

In kubernetes-sigs/prow, following PR #906 — "entrypoint: de-flake interrupt tests and fix SIGKILL-path data race" (merged as b9f26284d47bc7e7ba0b726c3b13144b4982ab29), probe both SIGKILL drain outcomes in pkg/entrypoint/run.go, especially Options.ExecuteProcess, gracefullyTerminate, and propagatedExitCode. Use TestExecuteProcess_KillPath in pkg/entrypoint/run_test.go as context. Start from the current default branch containing that merge and check whether later changes already cover this work.

Create temporary tests with explicitly synchronized subprocesses. Establish readiness after signal handlers are installed. Control the lifetime of descendants holding inherited stdout/stderr pipes through a handshake and explicit release, rather than a fixed sleep. Arrange cleanup for every fixture process on success and failure.

Exercise these paths:
1. The wrapped process ignores SIGINT/SIGTERM, survives the grace period, is SIGKILLed, and all pipe holders terminate promptly. Confirm the post-kill Wait completes, no drain-timeout diagnostic appears, and the propagated return code and marker both equal -1.
2. The wrapped process is SIGKILLed while a descendant demonstrably retains the pipe until the test releases it. Confirm entrypoint returns before that release, emits the drain-timeout diagnostic, and returns and writes -1. Use an independent generous deadline to detect hangs, then release the descendant and let fixture work finish.

Run the focused probes repeatedly under the Go race detector. Treat a reproducible ProcessState data race, an incorrect propagated code or marker, or failure to return while the descendant retains the pipe beyond the bounded drain window as a potential production bug. Classify each failure with evidence; fixture flakiness or mistaken expectations alone do not justify a production change.

Acceptance criteria: both outcomes were demonstrably exercised; report commands, results, and failure classifications. If a real production bug is demonstrated, apply only its minimal fix and report the precise reproducing case. If none is found, leave no repository files changed.

Scope: discard all temporary probe tests and fixtures. Deliver only a demonstrated production fix, if any. Preserve signal forwarding, the fixed one-second drain policy, and public configuration. Do not submit test-only fixes, retained regression tests, documentation changes, or unrelated refactoring. Report enough reproduction detail for the maintainer to decide whether to retain a regression test separately.
```
