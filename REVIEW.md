---
pr: kubernetes-sigs/prow#891
title: "chore(deps): bump github.com/fsnotify/fsnotify from 1.9.0 to 1.10.1"
head_sha: f3f1dd4d22db76bcee56acdd280947fb97192668
base: main
reviewed_at: 2026-09-01T16:37:43Z
verdict: approve
---

## Verdict

Approve. This is a direct-dependency-only update with a well-soaked tagged release. Prow's only fsnotify use is a non-recursive marker-file-directory watcher; the v1.10 changes are compatible with that usage and with Prow's Go 1.26.4 minimum.

## What this PR does

- Updates the direct Go module `github.com/fsnotify/fsnotify` from `v1.9.0` to `v1.10.1`.
- Updates the corresponding module checksums in `go.sum`.
- Takes upstream fixes for inotify, kqueue, and Windows watcher implementations.

## Findings

None.

## Checked

- Classified the diff as dependency-only: `go.mod` and `go.sum` only.
- Confirmed v1.10.1 is a tagged release from 2026-05-04; no fsnotify repository security advisories were reported.
- Confirmed the sole direct import is `pkg/pod-utils/wrapper/options.go:30`, where `WaitForMarkers` watches a directory for marker-file `Create` events.
- Confirmed the upstream Go 1.23 minimum is compatible with this repository's `go 1.26.4` directive.
- Reviewed v1.10.0 and v1.10.1 release notes; recursive-watch and `NoFollow` behavior changes do not affect Prow's use.
- Ran `go test ./pkg/pod-utils/wrapper` successfully.

## Open questions

None.
