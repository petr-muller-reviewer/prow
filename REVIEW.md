---
pr: kubernetes-sigs/prow#814
title: "chore(deps): bump gopkg.in/ini.v1 from 1.67.0 to 1.67.3"
head_sha: 6f73529cc59b14d030c7d2eef47770d90dbe3052
base: main
reviewed_at: 2026-07-28T23:46:58Z
verdict: approve
---

## What this PR does

- Dependabot bump of `gopkg.in/ini.v1` v1.67.0 → v1.67.3 in `go.mod`/`go.sum`.
- Dep-only change: no project code touched.

## Dependency analysis

- Direct dependency (no `// indirect` marker). Single importer: `pkg/sidecar/censor.go:37`, used in `loadIniData` (censor.go:611-623) to parse INI-formatted content for Prow's secret-censoring logic. Non-hot path, not attacker-controlled network input.
- New version v1.67.3 released 2026-06-08 (tagged, not a pseudo-version) — ~7 weeks old at review time. No freshness concern.
- Changelog v1.67.0→v1.67.3 (go-ini/ini): v1.67.1 fixed double-quoted value parsing with backslash continuations (#373); v1.67.2 applied `ValueMapper` to substituted reference values (#382); v1.67.3 optimized `Key.Strings` allocations (#385). No CVEs, no breaking API changes.
- Exposure: light. Single non-hot-path importer; behavior changes are narrow bug fixes unlikely to affect how Prow's censor logic parses secret files.

## Findings

None.

## Checked

- Diff scope: confirmed only `go.mod`/`go.sum` changed, no vendored or project source.
- Import surface: `gopkg.in/ini.v1` used only in `pkg/sidecar/censor.go`.
- Release provenance: tagged release via Go module proxy (`proxy.golang.org`), not a pseudo-version.
- Changelog between old and new versions via `gh api compare` and `gh release view` for each intermediate tag — no security fixes, no breaking changes.

## Open questions

None.
