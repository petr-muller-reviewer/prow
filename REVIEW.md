---
pr: kubernetes-sigs/prow#890
title: "chore(deps): bump google.golang.org/grpc from 1.82.1 to 1.83.1"
head_sha: 84b2c62092ef3ab688ba282a736f9d165787adde
base: main
reviewed_at: 2026-09-01T16:37:22Z
verdict: approve
---

## Verdict

Approve. This is a dependency-only update with no project-source changes. The grpc-go 1.83.1 security fixes are confined to xDS RBAC parsing, which Prow does not import; its passed PR CI and the release's transport memory-bound improvement leave no identified regression risk for Prow's grpc use.

## What this PR does

- Updates the direct `google.golang.org/grpc` requirement from `v1.82.1` to `v1.83.1` in `go.mod`.
- Refreshes the corresponding `go.sum` entries.
- Updates three indirect module selections: `cel.dev/expr`, the GCP OpenTelemetry detector, and the OpenTelemetry GCP detector.

## Findings

No findings.

## Checked

- The PR changes only `go.mod` and `go.sum`; `git diff --check` is clean.
- grpc-go `v1.83.1` is a tagged release published 2026-08-19. It was six days old when this PR merged and 13 days old at review time.
- The release fixes xDS RBAC header-matcher fail-open cases and bounds buffering overhead for small transport frames. Prow has no grpc xDS imports.
- Prow imports grpc from 11 Go files: Gangway's server and Google client, ResultStore client/writer, Pub/Sub reporter, generated Gangway stubs, and integration tests. These use ordinary client/server, TLS, metadata, status, and health APIs; Gangway and ResultStore are network-facing, but none use the changed xDS RBAC surface.
- PR CI passed: image build, integration, unit, race-detector, and lint jobs.

## Open questions

None.
