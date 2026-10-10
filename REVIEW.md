---
pr: kubernetes-sigs/prow#1005
title: "chore(deps): bump github.com/mattn/go-zglob from 0.0.7 to 0.0.8"
head_sha: bbf0a77272743da9bb75cc826fe6e5d0fad4f91d
base: main
reviewed_at: 2026-10-10T13:56:48Z
verdict: approve
---

# Review

## Verdict

Approve. This is a dep-only update to a tagged release with more than three weeks of soak time and no OSV advisories for the bumped module. The glob behavior change reaches Prow's sidecar censor rules, so deployed `exclude_directories` patterns ending in `/**` should be checked because nested paths will now match.

## What this PR does

- Updates direct dependency `github.com/mattn/go-zglob` from v0.0.7 to v0.0.8 in `go.mod` and refreshes its checksums in `go.sum`.
- Adds recursive matching for a trailing `**` that is a whole path component; `**.go` and `foo**` remain non-recursive.
- Changes matching behavior at existing `zglob.Match` call sites in config update filtering and sidecar artifact censorship without changing Prow source code.

## Findings

No findings.

## Checked

- The diff contains only `go.mod` and `go.sum` changes; there are no project-code edits.
- The Go module proxy dates v0.0.8 to 2026-09-16. GitHub has no release notes for the tag; the upstream comparison contains the globstar behavior fix and its tests.
- OSV reports no advisories for v0.0.7 or v0.0.8.
- `govulncheck ./...` reports the same seven module-only findings at base and head: GO-2023-1901, GO-2026-5932, GO-2026-6603, GO-2026-6610, GO-2026-6611, GO-2026-6612, and GO-2026-6617. None involve go-zglob or have called-function traces.
- The dependency is imported in `pkg/plugins/updateconfig/updateconfig.go:31` and `pkg/sidecar/censor.go:34`; matching occurs at `pkg/plugins/updateconfig/updateconfig.go:296` and `pkg/sidecar/censor.go:168,177`.
- Checked-in glob examples use `**/` followed by a final pattern component, such as `dir/**/*.yaml` and `/usr/**/*`; they do not exercise the newly changed trailing-`**` form.

## Open questions

- Do deployed sidecar configurations use `exclude_directories` patterns ending in `/**`? In v0.0.8 those rules will exclude nested descendants too, so those files will not be censored.

## Dependency followups

No opportunities identified for `github.com/mattn/go-zglob` v0.0.7→v0.0.8; it is the only module changed. The upstream change adds trailing-`**` matching behavior that existing `zglob.Match` call sites already expose to runtime patterns. Prow has no deprecated or superseded API calls or checked-in patterns that need migration, so there is no code handoff.
