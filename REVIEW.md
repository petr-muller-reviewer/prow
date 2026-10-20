---
pr: kubernetes-sigs/prow#882
title: "chore(deps): bump cloud.google.com/go/auth from 0.20.0 to 0.23.2"
head_sha: 9dd7380b6b839fad9f20f7d909355a4d5403061c
base: main
reviewed_at: 2026-09-01T18:02:52Z
verdict: approve
---

## Verdict

Approve. This is a dependency-only direct-module bump with a 12-day-old tagged release, no API change in Prow, and no upstream change that intersects Prow's sole `credentials.DetectDefault` use.

## What this PR does

- Updates the direct production dependency `cloud.google.com/go/auth` from `v0.20.0` to `v0.23.2` in `go.mod`.
- Updates only the two matching module checksums in `go.sum`.
- Takes upstream regional access-boundary and X.509 workload-identity support, an ID-token double-impersonation fix, and HTTP telemetry status tagging.
- Includes upstream OpenTelemetry test-flake fixes from `v0.23.1` and `v0.23.2`.

## Findings

No findings.

## Checked

- Classification: dependency-only; PR diff is limited to `go.mod` and `go.sum`.
- `cloud.google.com/go/auth v0.23.2` is a tagged release from 2026-08-20 (12 days old); it is not a pseudo-version.
- The module is direct, declares Go 1.25 at both old and new versions, and is compatible with Prow's Go 1.26.4 requirement.
- Prow imports it once at `pkg/io/credentials.go:22`, using `credentials.DetectDefault` for explicit GCS credential files and scopes.
- The credential helper is reached by GCS storage and Cloud Build setup, making it security-sensitive but with a single contained import surface.
- Upstream release notes from `v0.20.0` through `v0.23.2` contain no published CVE/security advisory and no change to Prow's `DetectDefault` call pattern.
- PR #882 was merged on 2026-08-25; this is a retrospective review.

## Open questions

None.
