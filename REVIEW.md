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

## Dependency followups

Examined on 2026-10-20: no concrete opportunities identified; no items or handoff prompts.

- Scope: direct Go module `cloud.google.com/go/auth v0.20.0 → v0.23.2`, including every intermediate release. PR #882 merged as `16d10b5cc4e4a965f9795ddb1b1ed05676185f71` on 2026-08-25.
- Baseline: compared the PR head `9dd7380b6b839fad9f20f7d909355a4d5403061c` against its original base `c55704ffb386ddc489728f5cd406f701525059b2`. The merge-base with current `upstream/main` is already the PR head, so it cannot represent the original bump.
- Transitive scope: no other module versions changed in Prow's manifests or checksums. The auth module raises several dependency minimums, but Prow's original base already pins those dependencies at equal or newer versions; there is no transitive bump attributable to this PR.
- Usage: the sole auth import is `pkg/io/credentials.go:22`; `credentials.DetectDefault` at line 32 loads the supplied credential file and scopes. Its callers are `pkg/io/opener.go:144` (GCS) and `pkg/googlecloudbuild/client/client.go:96` (Cloud Build). No new deprecation or replacement applies to those calls.
- Regional access-boundary changes are implemented inside the library. Prow has no matching boundary configuration or custom implementation to replace; enabling a new regional restriction would require a concrete deployment requirement.
- X.509 workload identity adds Enterprise Certificate Proxy support through credential configuration. Prow already delegates credential loading to the library, and no X.509/ECP configuration or hand-written certificate integration was found to migrate.
- HTTP response-status telemetry is automatic upstream behavior. Prow does not directly construct auth HTTP transports or duplicate this instrumentation. The ID-token impersonation fix affects an API Prow does not call, and the two patch releases only repair upstream tests.
- Sources: [auth changelog at v0.23.2](https://github.com/googleapis/google-cloud-go/blob/auth/v0.23.2/auth/CHANGES.md), [regional access-boundary implementation](https://github.com/googleapis/google-cloud-go/commit/fadb6c764dbe869fca736d9f0446b5010f21d2f2), [X.509 ECP implementation](https://github.com/googleapis/google-cloud-go/commit/5fa474a10a3863a1bb34ecb27540e8a574d7a736).
