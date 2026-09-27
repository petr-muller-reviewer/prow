---
pr: kubernetes-sigs/prow#979
title: "chore(deps): bump github.com/bombsimon/logrusr/v4 from 4.1.0 to 4.2.0"
head_sha: a9b50e53f608106db0cfe39efe023d68b4087e2d
base: main
reviewed_at: 2026-09-27T11:13:47Z
verdict: approve
---

# Review

## Verdict

Approve.

This is a dep-only, direct Go-module update with adequate release soak time and no known OSV advisories at either version. The one substantive upstream change corrects serialization of `error` values in structured controller-runtime log fields; Prow reaches the adapter only through `pkg/logrusutil`, and its package tests pass at the PR head.

## What this PR does

- Updates the direct `github.com/bombsimon/logrusr/v4` requirement from `v4.1.0` to the tagged `v4.2.0` release.
- Refreshes only the corresponding two module checksums; no Prow source, configuration, or generated code changes.
- Takes upstream's fix for structured `error` log fields being marshaled incorrectly.

## Findings

None.

## Resolved

None.

## Checked

- `v4.2.0` was created 2026-08-28, 30 days before review; it is a tagged release, not a pseudo-version.
- OSV returns no advisories for `github.com/bombsimon/logrusr/v4` at either `v4.1.0` or `v4.2.0`.
- Prow imports the module only at `pkg/logrusutil/logrusutil.go:25`, where `logrusr.New` supplies controller-runtime's Logrus bridge; `ComponentInit` is used by 32 production commands.
- The upstream `v4.1.0...v4.2.0` code change adds `error` to the adapter's native structured-field types, preserving useful error output without changing request, auth, secret-censoring, or transport behavior.
- `go test ./pkg/logrusutil` passes. `govulncheck` was unavailable in this environment.

## Open questions

None.

## Dependency followups

No improvement opportunities identified. Examined `github.com/bombsimon/logrusr/v4` `v4.1.0` → `v4.2.0`; the PR changes no other selected Go module version. The release fixes formatting of structured `error` fields but adds no replacement or deprecated API for Prow's sole `logrusr.New` call at `pkg/logrusutil/logrusutil.go:56`. Upstream's dependency version increases are already satisfied by Prow's selected versions.
