---
pr: kubernetes-sigs/prow#962
title: "config: reject partial job config directory walks"
head_sha: e8a95235cc1bea0d439ee2c93d389af63aad510b
base: main
reviewed_at: 2026-09-22T20:26:50Z
verdict: approve
---

## Verdict

approve — Directory-backed job configuration now fails atomically when a walk callback reports an error, preserving the last known-good configuration instead of publishing an incomplete job set. The change is small, uses the existing aggregate-error path, and has focused regression coverage.

## What this PR does

- Routes `ReadJobConfig` through a private helper that accepts a walker, preserving the exported API while enabling deterministic tests.
- Adds callback-reported `filepath.Walk` errors to the existing load-error aggregate, while continuing the walk to report further errors.
- Rejects the assembled partial `JobConfig` if any walk error occurred.
- Adds coverage for a valid job config loaded before a simulated git-sync checkout disappearance.

## Findings

No findings.

## Checked

- `pkg/config/config.go:1731-1785`: a callback error is wrapped with its path, added to `errs`, and therefore causes `utilerrors.NewAggregate` to reject the partial result.
- `pkg/config/config.go:1700-1704`: walker injection is private; production behavior still uses `filepath.Walk` and the public API is unchanged.
- `pkg/config/config_test.go:3692-3713`: the test reaches a valid YAML file before injecting a callback error and checks error identity with `errors.Is`.
- `pkg/config/agent.go:371-374`: a failed reload is not installed; the config agent retains its prior configuration.
- `go test ./pkg/config -run 'TestReadJobConfig(RejectsWalkErrors|ProwIgnore)$' -count=1` passed.

## Open questions

None.
