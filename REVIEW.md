---
pr: kubernetes-sigs/prow#935
title: "chore(deps): bump github.com/sirupsen/logrus from 1.10.1 to 1.10.2"
head_sha: 47985155847ea34b7e0e50007b2311562b26321f
base: main
reviewed_at: 2026-09-14T21:25:40Z
verdict: approve
---

## Verdict

approve — This dependency-only patch updates Logrus to a 19-day-old tagged maintenance release with no functional changes. Its sole upstream change is a test-dependency update that removes legacy `gopkg.in/yaml.v3` from Logrus's dependency graph; it does not affect Prow's runtime Logrus API use.

## What this PR does

- Updates the direct Go module `github.com/sirupsen/logrus` from `v1.10.1` to `v1.10.2` in `go.mod`.
- Refreshes the two corresponding module checksums in `go.sum`.
- Takes Logrus's `stretchr/testify` v1.12.1 test-dependency update, which removes its legacy `gopkg.in/yaml.v3` dependency.

## Findings

None.

## Checked

- Classification: dependency-only; only `go.mod` and `go.sum` change.
- Freshness/provenance: Logrus v1.10.2 is a tagged GitHub release published 2026-08-25 (19 days before review), not a pseudo-version.
- Upstream comparison `v1.10.1...v1.10.2`: four release-preparation/dependency commits; no functional changes or documented CVE/security fix.
- Direct usage: Logrus is Prow's logging dependency, imported by 381 Go files (257 non-test) across core controllers, webhooks, plugins, GitHub handling, and command binaries. The exposure is broad but limited to observability APIs; upstream's test-only dependency change does not intersect it.

## Open questions

None.
