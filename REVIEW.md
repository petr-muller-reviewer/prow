---
pr: kubernetes-sigs/prow#935
title: "chore(deps): bump github.com/sirupsen/logrus from 1.10.1 to 1.10.2"
head_sha: 654aa07184529556e783513a097c3c7b490be3ba
base: main
reviewed_at: 2026-11-08T00:11:07Z
verdict: approve
refresh_log:
  - at: 2026-11-08T00:11:07Z
    old_sha: 47985155847ea34b7e0e50007b2311562b26321f
    new_sha: 654aa07184529556e783513a097c3c7b490be3ba
    summary: "Rebase onto newer main (no change to PR content, identical patch-id); approved by petr-muller and merged as e210af8."
---

## Verdict

approve — This dependency-only patch updates Logrus to a 19-day-old tagged maintenance release with no functional changes. Its sole upstream change is a test-dependency update that removes legacy `gopkg.in/yaml.v3` from Logrus's dependency graph; it does not affect Prow's runtime Logrus API use.

## What this PR does

- Updates the direct Go module `github.com/sirupsen/logrus` from `v1.10.1` to `v1.10.2` in `go.mod`.
- Refreshes the two corresponding module checksums in `go.sum`.
- Takes Logrus's `stretchr/testify` v1.12.1 test-dependency update, which removes its legacy `gopkg.in/yaml.v3` dependency.

Since previous review:

- The PR was rebased from `4798515` onto newer `main` (`94b6caf`), producing head `654aa07`; the PR's own diff is byte-identical (`go.mod`/`go.sum`, 3 lines, same `git patch-id` `dd3ae7e`). The larger `4798515..654aa07` git range is unrelated upstream history (the `#936` `/hack/tools` Logrus bump and the `#930` Tide fix), not PR content.
- `petr-muller` approved at 2026-09-14T21:31:13Z; the PR merged at 2026-09-14T21:33:18Z as `e210af8`, whose `go.mod` carries `github.com/sirupsen/logrus v1.10.2`.
- All required checks passed on the merged head (`pull-prow-unit-test`, `-integration`, `-verify-lint`, `-image-build-test`, `tide`, `EasyCLA`).

## Findings

None.

## Checked

- Classification: dependency-only; only `go.mod` and `go.sum` change.
- Freshness/provenance: Logrus v1.10.2 is a tagged GitHub release published 2026-08-25 (19 days before review), not a pseudo-version.
- Upstream comparison `v1.10.1...v1.10.2`: four release-preparation/dependency commits; no functional changes or documented CVE/security fix.
- Direct usage: Logrus is Prow's logging dependency, imported by 381 Go files (257 non-test) across core controllers, webhooks, plugins, GitHub handling, and command binaries. The exposure is broad but limited to observability APIs; upstream's test-only dependency change does not intersect it.

## Open questions

None.

## Dependency followups

Run 2026-11-08T00:13:53Z against PR #935 (merged 2026-09-14T21:33:18Z as `e210af8`). No improvement opportunities identified.

- Modules examined: `github.com/sirupsen/logrus` `v1.10.1` → `v1.10.2` — direct, and the only version change in the PR (`go.mod` one line; `go.sum` two checksum lines for the new version, no other module's version moved).
- Changelog range `v1.10.1...v1.10.2` (4 upstream commits): a `github.com/stretchr/testify` v1.12.1 bump, its merge, and two changelog/release-prep merges. The v1.10.2 release note says "There are no functional changes in this release", and `CHANGELOG.md` lists only the testify update (removing Logrus's legacy `gopkg.in/yaml.v3`).
- Reason: the range contains no deprecations, new APIs, improved replacements, or behavior/default changes affecting Prow. The sole delta is a test-only dependency of Logrus, while Prow's 381 importing files (257 non-test) use only stable runtime logging APIs.
