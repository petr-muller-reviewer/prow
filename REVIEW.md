---
pr: kubernetes-sigs/prow#1048
title: "chore(deps): bump cloud.google.com/go/secretmanager from 1.22.0 to 1.23.0"
head_sha: cd5a9072bcef07fb98cbb7d7d07318fa52586991
base: main
reviewed_at: 2026-11-02T17:45:00Z
verdict: approve
---

# Review

## Verdict

Safe to bump now; the only caution is that v1.23.0 is 11 days old.

Dep-only change: `cloud.google.com/go/secretmanager` v1.22.0 -> v1.23.0 in `go.mod` and two `go.sum` lines. The release adds opt-in gax client tracing/logging to the generated clients and regenerates protos; the exported `apiv1` API is unchanged, our root module already requires the same or newer versions of everything the new module asks for, and OSV/govulncheck show no vulnerability delta between base and head. We use the package in a single wrapper (`cmd/webhook-server/secretmanager`) that calls only stable RPC methods, so the change is telemetry-only and off by default for us.

## What this PR does

- Bumps the direct dependency `cloud.google.com/go/secretmanager` from v1.22.0 to v1.23.0 (`go.mod:9`), with the matching `go.sum` hash updates; no other files change.
- Takes googleapis/google-cloud-go release `secretmanager/v1.23.0` (2026-10-22), whose only substantive change is `feat: Update API sources and regenerate (#20728)`.
- Pulls in generated-client telemetry wiring: tracing and logging call options gated behind `GOOGLE_SDK_GO_EXPERIMENTAL_TRACING` / `GOOGLE_SDK_GO_EXPERIMENTAL_LOGGING`, replacing the per-RPC `resource_name` telemetry context blocks.
- Regenerates the Secret Manager protobufs (field-behavior annotation and descriptor churn only); no message fields or client methods used by Prow change.
- Adds no new transitive dependencies to this repo: the module's raised requires (iam v1.12.0, gax v2.26.2, api v0.288.0) are all at or below what the root `go.mod` already pins.

## Findings

No findings.

## Checked

- Classification: dep-only, 2 files / 3 additions / 3 deletions; no project code, config, docs, or tests touched, so no standard code review applies.
- Release provenance: v1.23.0 is a real tag (`refs/tags/secretmanager/v1.23.0`), not a pseudo-version; published by `release-please[bot]` on 2026-10-22T21:06Z.
- Freshness: 11 days old at review time (2026-11-02), outside the too-fresh window but still inside two weeks.
- Usage: direct import only at `cmd/webhook-server/secretmanager/secretmanager.go:23-24`; `go mod why` chain is `cmd/webhook-server/secretmanager -> cloud.google.com/go/secretmanager/apiv1`; instantiated at `cmd/webhook-server/main.go:151` and consumed through `cmd/webhook-server/clients.go:36`. `apiv1beta2` is not imported.
- Exported API of `apiv1` is identical between v1.22.0 and v1.23.0 (same methods and `NewClient` signature); only the telemetry call options were added.
- OSV: no advisories for `cloud.google.com/go/secretmanager` at v1.22.0 or v1.23.0.
- govulncheck module and symbol scans at both merge-base and PR head: identical result sets. Four pre-existing advisories in unrelated modules (GO-2023-1901 tektoncd/pipeline — 32 called, 16 imported-only; GO-2026-5932 x/crypto — imported-only; GO-2026-6860 and GO-2026-6866 x/net — imported-only). None involve secretmanager, so the bump neither fixes nor introduces reachable vulnerabilities.
- Changelog diff reviewed at the module level (proxy zips of both versions): only generated client files, regenerated protos, `internal/version.go`, and the module's own `go.mod`/`go.sum` differ.
- Logging behavior checked in gax v2.26.2: `recordActionableLog` fires only on terminal failure and emits metadata (RPC method, error type/status/message, server address) — no request or response payloads, so secret values are not logged.
- Build: `GOEXPERIMENT=jsonv2 go vet ./cmd/webhook-server/...` passes at head. The jsonv2 experiment is pre-existing (`pkg/config/zz_generated.deepcopy.go:25` imports `encoding/json/jsontext`), unrelated to this bump.

## Open questions

- Is 11 days of soak acceptable, or would you rather wait until v1.23.0 is past two weeks? Nothing in the diff argues for holding it.

## Dependency followups

No opportunities identified.

- `cloud.google.com/go/secretmanager` v1.22.0 -> v1.23.0 (direct): the only upstream change in range is `feat: Update API sources and regenerate (#20728)` — opt-in gax tracing/logging wiring plus regenerated protos, with no exported field, method, or constructor changes; nothing we call is deprecated or superseded, and no new API replaces hand-rolled code of ours.
- Transitive churn: none. Only the two `go.sum` lines for secretmanager move; the module's raised requires (iam v1.12.0, gax v2.26.2, api v0.288.0) are already met or exceeded by the root `go.mod`.
- Usage examined: `cmd/webhook-server/secretmanager/secretmanager.go` (all six wrapper RPCs) and its consumers in `cmd/webhook-server/clients.go` and `cmd/webhook-server/main.go`. The new telemetry is env-var gated (`GOOGLE_SDK_GO_EXPERIMENTAL_TRACING` / `_LOGGING`) and needs no call-site change.
