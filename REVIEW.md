---
pr: kubernetes-sigs/prow#896
title: "chore(deps): bump github.com/clarketm/json from 1.13.4 to 1.17.1"
head_sha: 6098ff515ed14120f0c0613762b0a9b1578c37e3
base: main
reviewed_at: 2026-09-01T09:34:11Z
verdict: approve
---

## Verdict

Approve. This is a direct but narrowly used JSON encoder bump; the upstream changes are well-soaked and the only affected package tests successfully.

## What this PR does

- Updates `github.com/clarketm/json` from `v1.13.4` to `v1.17.1` in `go.mod` and `go.sum`.
- Retains the existing direct dependency used by `pkg/genyaml`.
- Picks up JSON decoder nesting-depth protection, encoder cycle detection, and encoding fixes.
- Adds zero-value struct `omitempty` behavior used by this project's custom JSON fork.

## Findings

No findings.

## Checked

- Dependency-only diff: `go.mod` and `go.sum`; no project source changes.
- `v1.17.1` is a non-pseudo tagged version released 2021-09-14, with ample soak time.
- Direct import surface is one package: `pkg/genyaml`; `marshal` uses `json.Marshal` at `pkg/genyaml/genyaml.go:164`, and diagnostics use `json.MarshalIndent` at `pkg/genyaml/genyaml.go:460`.
- Upstream range contains decoder nesting-depth protection, cycle detection, corrected encoding behavior, and zero-value struct `omitempty` support; no published repository advisory was returned.
- `go test ./pkg/genyaml` passed.
- PR CI checks passed before merge.

## Open questions

None.
