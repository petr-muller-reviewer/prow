---
pr: kubernetes-sigs/prow#899
title: "chore(deps): bump github.com/evanphx/json-patch from 5.9.0+incompatible to 5.9.11+incompatible"
head_sha: 9058f0eb9be5e19778b614d13ecbe3d059f4eabd
base: main
reviewed_at: 2026-09-01T11:06:51Z
verdict: approve
---

## What this PR does

- Dependabot bump of `github.com/evanphx/json-patch` from `v5.9.0+incompatible` to `v5.9.11+incompatible`.
- Dep-only change: touches only `go.mod` (1 line) and `go.sum` (2 lines). No vendoring in this repo, no project code changed.
- Direct dependency; unrelated indirect siblings (`github.com/evanphx/json-patch/v5`, `gopkg.in/evanphx/json-patch.v4`) are untouched by this bump.

## Dependency analysis

**Freshness**: new version `v5.9.11+incompatible` tagged 2025-01-28 (commit `84a4bb1`, real git tag, not a pseudo-version). Well over a year old as of review date — fine, no soak-time concern.

**Usage**: direct dep. Imported in exactly one place: `pkg/pjutil/abort.go:26,135`, via `jsonpatch.CreateMergePatch` to build a merge patch when aborting in-flight ProwJobs. Operates on Prow's own internal `ProwJob` objects, not untrusted external input. Light exposure, non-sensitive path (no auth/crypto/exec/deserialization of external data).

**Changelog (v5.9.0 → v5.9.11)**: three substantive upstream commits — export previously-unexported error variables (`errBadJSONDoc`, `errBadJSONPatch`), drop a stale `gopkg.in` reference in the v5 module declaration, remove the unmaintained `github.com/pkg/errors` dependency. No CVE/security fix, no behavior change to `CreateMergePatch`.

**Take**: safe to take now. Old release, non-behavioral changelog, single light-touch non-sensitive call site.

## Findings

None.

## Checked

- Diff scope: `go.mod`/`go.sum` only, no vendored/generated files, no project code.
- Only import site (`pkg/pjutil/abort.go`) and how `CreateMergePatch` is used there.
- Upstream commit range v5.9.0..v5.9.11 for behavior/security-relevant changes.
- Release age/provenance via `proxy.golang.org` (real tag, not pseudo-version).

## Open questions

None.
