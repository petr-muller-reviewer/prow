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
