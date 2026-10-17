---
pr: kubernetes-sigs/prow#874
title: "chore(deps): bump github.com/felixge/fgprof from 0.9.1 to 0.9.5"
head_sha: 76b0e291246201e190f5bd5a36e4a739a432f8ce
base: main
reviewed_at: 2026-09-01T18:53:42Z
verdict: approve
---

## Verdict

Approve. This is a dependency-only update of a direct profiling dependency to a tagged release from 2024-08-30; Prow's sole integration is `fgprof.Handler()` on its dedicated profiling server, and the upstream changes do not alter that API.

## What this PR does

- Updates `github.com/felixge/fgprof` from `v0.9.1` to `v0.9.5` in `go.mod`.
- Regenerates `go.sum` for fgprof and its updated transitive module metadata.
- Takes upstream profiler overhead and profile-metadata improvements.
- Includes the upstream workaround for the Go 1.23 goroutine-profile regression.

## Findings

None.

## Checked

- Classification: dependency-only; only `go.mod` and `go.sum` changed.
- Freshness/provenance: `v0.9.5` is a non-pseudo tagged release published 2024-08-30, with ample soak time.
- Usage: direct dependency, imported only by `pkg/pjutil/pprof/pprof.go:29`; its handler is registered at `pkg/pjutil/pprof/pprof.go:55` on the profiling server. The helper is shared by 18 binaries, but the imported surface is one diagnostic handler.
- Upstream `v0.9.1...v0.9.5`: profiling/STW-overhead reductions, more accurate profile timing metadata, line-number support, `google/pprof` update, and the Go 1.23 workaround. No changed behavior affects Prow's `fgprof.Handler()` use.
- No security advisory or CVE was announced in the upstream release notes or commit range.
- `git diff --check` passes.

## Open questions

None.

## Dependency followups

No opportunities identified. Examined `github.com/felixge/fgprof v0.9.1 → v0.9.5`: the upstream changes improve profiler internals and preserve the `Handler()` API used at `pkg/pjutil/pprof/pprof.go:55`, so they require no project-side migration. The bump also updates fgprof's `github.com/google/pprof` module requirement from `v0.0.0-20200615235658-03e1cf38a040` to `v0.0.0-20240227163752-401108e1b7e7`, but Prow already selected `v0.0.0-20250403155104-27863c87afa6` in the PR base; no effective transitive upgrade or Prow-owned pprof call site is introduced.
