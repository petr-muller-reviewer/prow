---
pr: kubernetes-sigs/prow#1004
title: "chore(deps): bump cloud.google.com/go/secretmanager from 1.21.0 to 1.22.0"
head_sha: d9f87711dab915a46a756db915979f7814e46eaf
base: main
reviewed_at: 2026-10-10T13:56:31Z
verdict: approve
---

# Review

## Verdict

Approve. This is a dependency-only bump to a tagged release with no OSV advisory for either version. The release is past the two-week freshness window, and its only documented change is updating supported Go versions; Prow already requires Go 1.26.4.

## What this PR does

- Updates the direct `cloud.google.com/go/secretmanager` requirement from `v1.21.0` to `v1.22.0`.
- Replaces the module's two `go.sum` checksums for the new version.
- Changes no project source or configuration files.

## Findings

None.

## Checked

- The diff contains only `go.mod` and `go.sum`; this is dep-only, so there are no project-code changes to review.
- The module is direct and is imported in one project Go file/package, `cmd/webhook-server/secretmanager/secretmanager.go`. That wrapper creates secrets, writes versions, and reads secret values, so the use is sensitive but has a light import surface.
- The Go module proxy dates `v1.22.0` to 2026-09-24; it is a tagged release and was 16 days old at review time.
- OSV returned no advisories for `v1.21.0` or `v1.22.0`.
- `govulncheck` at the merge base and PR head emitted the same repo-wide IDs: `GO-2023-1901`, `GO-2026-5932`, `GO-2026-6603`, `GO-2026-6610`, `GO-2026-6611`, `GO-2026-6612`, and `GO-2026-6617`. No finding trace involved Secret Manager, and the bump added or fixed no finding for this module.
- The `v1.22.0` changelog says “Update supported go versions”; the module raises its Go requirement from 1.25 to 1.26. Prow requires Go 1.26.4. The module's updated minimums for gRPC and `golang.org/x` dependencies are below the versions Prow already selects, so they do not move Prow's effective dependency versions.

## Open questions

None.

## Dependency followups

No actionable follow-up was identified for `cloud.google.com/go/secretmanager` `v1.21.0` → `v1.22.0`. Its changelog only updates supported Go versions; it contains no API deprecation, replacement, feature, or behavior change applicable to Prow's usage. The bump adds no transitive version changes to Prow's manifest or checksum file; the higher dependency minimums in Secret Manager's own `go.mod` are already superseded by versions Prow selects. No handoff prompts were produced.
