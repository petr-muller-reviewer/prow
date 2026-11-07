---
pr: kubernetes-sigs/prow#836
title: "chore(deps): bump github.com/go-git/go-git/v5 from 5.19.1 to 5.19.2"
head_sha: aeb64c10509fa319b7db2e3c730f1f15c2f87608
base: main
reviewed_at: 2026-08-18T10:55:33Z
verdict: approve
---

## Summary

Dep-only bump: `github.com/go-git/go-git/v5` v5.19.1 -> v5.19.2. Only `go.mod`/`go.sum` changed (3 lines). PR already merged 2026-08-11.

## Dependency analysis

- Release: v5.19.2 tagged 2026-07-29, ~20 days before merge. Real tag, not a pseudo-version. Fine on freshness.
- Direct dependency (`go.mod:33`, no `// indirect`).
- Import surface (excluding vendor): `pkg/plugins/testfreeze/checker/checker.go` (+ its test/fake) using `git.NewRemote`/`ListRefs` against in-memory storage (read-only ref listing); `test/integration/internal/fakegitserver/fakegitserver.go` using `PlainOpen` (test-only fake server). No `Clone`/`Fetch`/`Worktree` checkout usage, no direct `SetRef`/`RemoveRef` calls.
- Changelog v5.19.1..v5.19.2 highlights: `storage: dotgit, reject path traversal in reference names` (#2254, guards `SetRef`/`Ref`/`RemoveRef` against malicious ref names from remotes) and `worktree, make the filesystem wrapper a symlink-safe boundary` (#2277). Both are real security hardening upstream but land on code paths (write-side ref storage, worktree checkout) that prow does not exercise — only read-only `ListRefs` and a test-only fake server. Also several transitive `golang.org/x/{crypto,net,text}` `[SECURITY]` bumps pulled in by go-git's own deps.

## Findings

(none — dep-only bump, low-risk, no code review needed)

## Checked
- Classified PR as dep-only (only go.mod/go.sum changed).
- Verified go-git is a direct dependency and its import surface in prow (4 files, read-only ref listing + test fixture).
- Read upstream changelog/commit range between v5.19.1 and v5.19.2; confirmed no exposure to the security-relevant fixes (path traversal in ref storage, worktree symlink escape) since prow doesn't use those code paths.
- Confirmed release age (~20 days) is past the soak-time threshold.

## Open questions
(none)

## Dependency followups

No improvement opportunities identified.

- Examined: `github.com/go-git/go-git/v5` v5.19.1 → v5.19.2 (only module whose lines changed in `go.mod`/`go.sum`; PR merged as `08ac810c3d783aa8e58e69dacf097b4bbbb14f25`).
- Changelog v5.19.1..v5.19.2 covers two code changes — `worktree` symlink-safe filesystem boundary (#2277) and Windows backslash handling in the worktree index (#2262) — plus `storage/dotgit` reference-name containment hardening (#2254) and the new `plumbing.ReferenceName.IsSafe`. Our only call sites are `git.NewRemote`/`Remote.List` with `ref.Name().IsBranch()`/`Short()` filtering against a hardcoded `kubernetes/kubernetes` remote (`pkg/plugins/testfreeze/checker/checker.go:162`, `:314`) and test-only `PlainOpen`/`Head`/`Log` in `test/integration/internal/fakegitserver/fakegitserver.go:214`. We use no worktree/checkout, no write-side ref storage, and `IsBranch()` already narrows names to `refs/heads/`, so none of the new APIs or hardening apply.
- Transitive `golang.org/x/{crypto,net,text}` security bumps exist only in go-git's own `go.mod`; prow's `go.sum` changed just the two go-git lines, so our module graph versions are unchanged — nothing to adopt or re-pin.

No followup items accepted, no handoff prompts.
