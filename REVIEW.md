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

## Dependency followups

Examined on 2026-10-21. PR #899 merged on 2026-08-26 as `c395925e86cc5ba81c6d07230322129616aeddf6`.

No concrete improvement opportunities were identified; no followup items or handoff prompts were created.

- **Module range:** direct Go dependency `github.com/evanphx/json-patch`, `v5.9.0+incompatible` → `v5.9.11+incompatible`. The complete manifest/checksum diff contains no transitive module version changes; `github.com/evanphx/json-patch/v5` and `gopkg.in/evanphx/json-patch.v4` are unchanged.
- **Our usage:** the sole import is `pkg/pjutil/abort.go:26`; `PatchProwjob` calls `jsonpatch.CreateMergePatch` at line 135 on JSON marshaled from two ProwJobs and already wraps errors with `fmt.Errorf` and `%w`.
- **Reason:** the legacy module's `merge.go`, including `CreateMergePatch`, is unchanged across the range. Replacing `github.com/pkg/errors` wrapping in JSON Patch operations provides no useful call-site migration here. The newly exported `ErrBadJSONDoc`/`ErrBadJSONPatch` symbols affect the separate `/v5` module; documentation and `/v5` dependency changes also provide no applicable improvement. No relevant deprecations, replacement APIs, or new features were found.
- **Source:** [complete upstream v5.9.0…v5.9.11 comparison](https://github.com/evanphx/json-patch/compare/v5.9.0...v5.9.11), covering all eight intervening commits, cross-checked against the PR diff and Prow's import/call site.
