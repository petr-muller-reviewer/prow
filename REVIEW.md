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

## Dependency followups

Examined on 2026-10-21. No actionable opportunities identified; no items or handoff prompts.

- Scope: PR #891, merged as `da76fd4bf78649f471b25f7cd438d554c7626e7e`; direct Go dependency `github.com/fsnotify/fsnotify v1.9.0 → v1.10.1`, covering intermediate v1.10.0. Since the PR is already merged into the tracked base, used the PR-head commit's parent for the manifest comparison. Only fsnotify changed; no transitive version changes. Upstream's `golang.org/x/sys v0.13.0` requirement is unchanged.
- Usage: `pkg/pod-utils/wrapper/options.go:102` creates a watcher, line 107 closes it, line 119 adds one directory, and lines 132–139 consume events/errors and filter marker-file `Create` events. Entrypoint and sidecar call this helper.
- Reason: the releases provide backend fixes and an inotify performance improvement that apply automatically, plus a Go minimum increase already satisfied by Prow. They introduce no replacement API, deprecation, or useful new public feature requiring changes to this call site. Recursive-watch fixes concern an internal capability unavailable through the public API; the removed no-follow option was also internal. Neither justifies followup work here.
- The polling fallback at `pkg/pod-utils/wrapper/options.go:124` and lines 140–145 is not made redundant by these fixes. The separate `gopkg.in/fsnotify.v1` imports in `pkg/config/agent.go` and `pkg/flagutil/kubernetes_cluster_clients.go` are not bumped by this PR; migrating them is not unlocked by this version range.
- Sources: [v1.10.1 changelog, including v1.10.0](https://github.com/fsnotify/fsnotify/blob/v1.10.1/CHANGELOG.md), [complete upstream version comparison](https://github.com/fsnotify/fsnotify/compare/v1.9.0...v1.10.1).
