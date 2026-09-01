---
pr: kubernetes-sigs/prow#876
title: "chore(deps): bump cloud.google.com/go/storage from 1.56.0 to 1.65.0"
head_sha: 71779405cdfd3bb63a4890570991edf7d4272992
base: main
reviewed_at: 2026-09-01T18:48:20Z
verdict: approve
---

## Verdict

Approve. This is dependency-only churn. Storage v1.65.0 has adequate soak time, its Go 1.25 requirement is compatible with Prow's Go 1.26.4 declaration, and the intervening behavior changes do not overlap Prow's ordinary GCS object I/O surface in a concerning way.

## What this PR does

- Updates the direct GCS Storage client from `cloud.google.com/go/storage v1.56.0` to `v1.65.0`.
- Resolves Storage's newer Google Cloud, OpenTelemetry, monitoring, and generated-proto dependencies.
- Updates direct `cloud.google.com/go/secretmanager` from `v1.16.0` to `v1.20.0` and `google.golang.org/genproto` to a newer pseudo-version as part of module resolution.
- Changes only `go.mod` and `go.sum`; no Prow source, configuration, or tests change.

## Findings

None.

## Checked

- PR diff is dependency-only: 29 additions and 29 deletions across `go.mod` and `go.sum`.
- Storage v1.65.0 is a tagged 2026-08-19 release; it was six days old at merge and is 13 days old at review.
- Storage requires Go 1.25.0; Prow declares Go 1.26.4.
- Prow imports Storage in seven files, with production use concentrated in `pkg/io/opener.go` for GCS client construction, object reads/writes/listing, metadata updates, preconditions, and signed URLs.
- Intervening Storage releases add checksum/retry/writer and reader/download fixes plus APIs Prow does not call; no CVE/security advisory was identified in their release notes.
- Secret Manager remains used only by `cmd/webhook-server/secretmanager`; its intervening releases are API-source regeneration.
- `git diff --check` passes.

## Open questions

None.
