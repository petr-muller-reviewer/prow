---
pr: kubernetes-sigs/prow#950
title: "chore(deps): bump github.com/mattn/go-zglob from 0.0.6 to 0.0.7"
head_sha: 4776522984ed027ab42181ba2c880ba73d49df9d
base: main
reviewed_at: 2026-11-10T01:16:53Z
verdict: approve
refresh_log:
  - at: 2026-11-10T01:16:53Z
    old_sha: 4776522984ed027ab42181ba2c880ba73d49df9d
    new_sha: 4776522984ed027ab42181ba2c880ba73d49df9d
    summary: "No code changes; PR approved by petr-muller, labeled lgtm/approved, and merged by Tide at the reviewed SHA."
---

## Verdict

Approve. This is a manifest-only direct dependency bump. The sole upstream functional change fixes a Unix `fastwalk` checkptr failure; Prow invokes only `zglob.Match`, not the changed directory-walking path. Release provenance, module integrity, and PR CI are satisfactory. The PR has since been approved and merged at the reviewed SHA (`4776522`), so nothing in this review is outstanding.

## What this PR does

- Updates direct production dependency `github.com/mattn/go-zglob` from `v0.0.6` to `v0.0.7`.
- Updates the corresponding two checksums in `go.sum`.
- Takes an upstream Unix `fastwalk` fix that avoids unsafe overlay of a short directory-entry record.

Since previous review:
- No new commits: the PR head is still `4776522` (`0.0.6` → `0.0.7`, `go.mod`/`go.sum` only).
- `petr-muller` approved at `2026-09-22T11:41:04Z`; Prow applied `lgtm` and `approved` labels.
- Tide merged the PR at `2026-09-22T12:04:13Z`; the head branch was deleted. `kubevirt-bot` cross-referenced it on `2026-09-25T06:10:39Z`.

## Findings

No findings.

## Checked

- Diff is limited to `go.mod` and `go.sum`; no project code changes.
- `v0.0.7` is a tagged release from 2026-09-15, approximately six days old at review time; it is not a pseudo-version.
- Upstream delta is one bug fix plus a regression test: copy a directory-entry record before parsing it, preventing a `checkptr` failure in `fastwalk`.
- Prow directly imports the module in `pkg/sidecar/censor.go` and `pkg/plugins/updateconfig/updateconfig.go`, both only through `zglob.Match`; no Prow call site reaches `zglob.Glob` or `fastwalk`.
- No published GitHub security advisories were reported for the upstream repository.
- `go mod verify` passed; `git diff --check` was clean.
- PR CI passed: image build, integration, unit, nonblocking race, and lint.
- Post-review: the required approval/LGTM labels were applied and Tide merged the PR at the reviewed SHA, confirming the observations above.

## Open questions

None.

## Dependency followups

No opportunities identified. Examined `github.com/mattn/go-zglob` v0.0.6 → v0.0.7 (direct; the only module changed in this PR). The range contains a single functional commit, `392023f2 fastwalk: fix checkptr failure in parseDirEnt` (plus its merge `d55adcf4`), touching only `fastwalk/fastwalk_unix.go` and adding `fastwalk/fastwalk_dirent_test.go`; it introduces no deprecations, no new or changed exported API, and no default changes. Prow's only call sites are `zglob.Match` at `pkg/sidecar/censor.go:168`, `pkg/sidecar/censor.go:177`, and `pkg/plugins/updateconfig/updateconfig.go:296`, none of which reach `fastwalk` or any other API this release changes.
