---
pr: kubernetes-sigs/prow#819
title: "chore(deps): bump google.golang.org/protobuf from 1.36.10 to 1.36.11"
head_sha: 958edd01962a1eddedca906380cfe3f55de4c9f4
base: main
reviewed_at: 2026-08-05T11:33:57Z
verdict: approve
---

## Summary

Dependabot dep-only bump of `google.golang.org/protobuf` v1.36.10 -> v1.36.11. Diff limited to `go.mod` (1 line) and `go.sum` (2 lines). No project source changed.

## Dependency analysis

- Freshness: v1.36.11 tagged 2025-12-12 (real tag, not pseudo-version). ~8 months old at review time. Well soaked.
- Usage: direct dependency (no `// indirect`). Imported in 11 non-vendor files, concentrated in `pkg/gangway/*` (external-facing gRPC service accepting untrusted input), `pkg/resultstore/*`, `hack/tools/tools.go`, integration tests.
- Changelog (protocolbuffers/protobuf-go v1.36.10...v1.36.11): no CVE. Notable fix: `internal/impl: check recursion limit in lazy decoding validation` — closes a gap where lazily-unmarshaled messages/extensions could bypass the recursion-depth limit (10,000), relevant to `pkg/gangway` decoding untrusted gRPC input. Other fix (`internal/encoding/tag: use proto3 defaults if proto3`) addresses a `checkptr` fatal error under `-race`/`-asan`/`-msan` builds, not applicable to this project's build config. Remaining changes are internal test/flake fixes and edition/Kythe tooling irrelevant here.
- Exposure: light-to-moderate. Recursion-limit fix is a mild positive (defense-in-depth for gangway's external input), not a newly disclosed exploitable bug.

## Findings

(none — dep-only PR, no project code changed)

## Checked
- go.mod/go.sum diff is limited to the single module bump, no other churn.
- New version is a real tagged release, not a pseudo-version.
- No CVE or security advisory associated with the bump.
- Import surface confirmed direct and traced to gangway (external gRPC), resultstore, and test-only files.

## Open questions
(none)
