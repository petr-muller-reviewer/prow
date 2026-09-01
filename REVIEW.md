---
pr: kubernetes-sigs/prow#900
title: "chore(deps): bump github.com/GoogleCloudPlatform/testgrid from 0.0.123 to 0.0.176"
head_sha: 6913b7d77120370d6757e7f0e5ea77b7d7891a76
base: main
reviewed_at: 2026-09-01T00:50:49Z
verdict: approve
---

## Verdict

Approve. This dependency-only update is adequately soaked, remains API-compatible with Prow's usage, and passed the repository's unit, integration, image-build, lint, and race-detector checks.

## What this PR does

- Updates the direct `github.com/GoogleCloudPlatform/testgrid` dependency from `v0.0.123` to `v0.0.176`.
- Updates indirect dependencies `bitbucket.org/creachadair/stringset` from `v0.0.9` to `v0.0.11` and `github.com/fvbommel/sortorder` from `v1.0.1` to `v1.1.0`.
- Removes obsolete `go.sum` entries produced by dependency resolution.
- Changes no Prow source code.

## Findings

No findings.

## Checked

- `v0.0.176` is a tagged release from 2026-08-10, giving it 22 days of soak time at review.
- Testgrid is a direct dependency imported by 20 Prow files across 11 packages; 14 imports are in production files.
- Prow exercises Testgrid metadata models, JUnit parsing, configuration protobufs, and GCS utilities in artifact upload, reporting, Deck, and Spyglass paths.
- The 889-commit upstream range primarily adds configuration and ResultStore capabilities, extends JUnit metadata parsing, and updates GCS/config internals; the APIs Prow exercises remain compatible.
- The configuration protobuf changes are additive for Prow's usage.
- Prow does not directly import either updated indirect dependency.
- GitHub CI passed unit, integration, image-build, lint, and nonblocking race-detector jobs for head `6913b7d77120370d6757e7f0e5ea77b7d7891a76`.

## Open questions

None.
