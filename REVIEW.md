---
pr: kubernetes-sigs/prow#950
title: "chore(deps): bump github.com/mattn/go-zglob from 0.0.6 to 0.0.7"
head_sha: 4776522984ed027ab42181ba2c880ba73d49df9d
base: main
reviewed_at: 2026-09-21T23:22:20Z
verdict: approve
---

## Verdict

Approve. This is a manifest-only direct dependency bump. The sole upstream functional change fixes a Unix `fastwalk` checkptr failure; Prow invokes only `zglob.Match`, not the changed directory-walking path. Release provenance, module integrity, and PR CI are satisfactory.

## What this PR does

- Updates direct production dependency `github.com/mattn/go-zglob` from `v0.0.6` to `v0.0.7`.
- Updates the corresponding two checksums in `go.sum`.
- Takes an upstream Unix `fastwalk` fix that avoids unsafe overlay of a short directory-entry record.

## Findings

No findings.

## Checked

- Diff is limited to `go.mod` and `go.sum`; no project code changes.
- `v0.0.7` is a tagged release from 2026-09-15, approximately six days old at review time; it is not a pseudo-version.
- Upstream delta is one bug fix plus a regression test: copy a directory-entry record before parsing it, preventing a `checkptr` failure in `fastwalk`.
- Prow directly imports the module in `pkg/sidecar/censor.go` and `pkg/plugins/updateconfig/updateconfig.go`, both only through `zglob.Match`; no Prow call site reaches `zglob.Glob` or `fastwalk`.
- No published GitHub security advisories were reported for the upstream repository.
- `go mod verify` passed; `git diff --check` was clean.
- PR CI passed: image build, integration, unit, nonblocking race, and lint. Tide remains pending required approval/LGTM labels.

## Open questions

None.
