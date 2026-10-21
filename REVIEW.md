---
pr: kubernetes-sigs/prow#887
title: "chore(deps): bump github.com/djherbis/atime from 1.0.0 to 1.1.0"
head_sha: 1a3f21ed7967a7c26c7b04edc000a31320b33045
base: main
reviewed_at: 2026-09-01T16:56:22Z
verdict: approve
---

## Verdict

Approve. This dependency-only update moves `github.com/djherbis/atime` from `v1.0.0` to the long-soaked tagged `v1.1.0` release. Its substantive upstream change is Go-module and WebAssembly support; Prow's only import remains on native filesystem-atime retrieval, whose platform implementations did not change.

## What this PR does

- Updates the direct production requirement `github.com/djherbis/atime` from `v1.0.0` to `v1.1.0` in `go.mod`.
- Replaces the corresponding module and module-metadata checksums in `go.sum`.
- Does not modify Prow source, tests, configuration, or generated code.

## Findings

### Blocking

None.

### Should fix

None.

### Nits

None.

### Questions

None.

## Checked

- Classified the diff as dependency-only: only `go.mod` and `go.sum` changed.
- Confirmed `v1.1.0` is a tagged release published 2021-04-25, with no OSV advisories returned for this module.
- Reviewed the upstream `v1.0.0...v1.1.0` diff: the code change adds `atime_wasm.go`; the remaining substantive packaging change is `go.mod`.
- Confirmed the direct import surface is one file: `pkg/diskutil/diskutil.go:24`, where `GetATime` calls `atime.Stat` at line 47. No callers of `GetATime` exist outside that package.
- Ran `go test ./pkg/diskutil` successfully; the package contains no test files.

## Open questions

None.
