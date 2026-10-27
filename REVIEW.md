---
pr: kubernetes-sigs/prow#866
title: "chore(deps): bump cloud.google.com/go/cloudbuild from 1.22.0 to 1.32.0"
head_sha: 248712a806947b17c5b0b63425f18f561e6a1f87
base: main
reviewed_at: 2026-08-25T10:50:23Z
verdict: approve
---

## What this PR does

- Dependabot sweep bumping `cloud.google.com/go/cloudbuild` v1.22.0 → v1.32.0 (title dep), plus lockstep direct bumps of `cloud.google.com/go/pubsub` (v1.47.0→v1.50.1), `cloud.google.com/go/secretmanager` (v1.14.5→v1.16.0), `cloud.google.com/go/storage` (v1.50.0→v1.56.0), and `google.golang.org/api` (v0.233.0→v0.287.1).
- Large tail of indirect transitive bumps in the `cloud.google.com/go/*` / otel / gax-go family, moved in lockstep with the direct bumps.
- Only `go.mod` (49 lines) and `go.sum` (124 lines) touched — no project source changes.

## Findings

None.

## Checked

- Classification: dep-only (no code outside `go.mod`/`go.sum`), so no standard code review performed.
- Freshness: all five direct bumps are well past soak — cloudbuild (2026-07-13, ~6wk), google.golang.org/api (2026-07-07, ~7wk), pubsub (2025-09-04), secretmanager (2025-10-20), storage (2025-07-25). No pseudo-versions.
- cloudbuild usage: direct, `pkg/googlecloudbuild/client/{client.go,fake/fake.go}` + test. Compare `cloudbuild/v1.22.0...v1.32.0` on `googleapis/google-cloud-go` has zero `(cloudbuild)`-scoped conventional commits — routine proto/API-surface regen only, light/non-sensitive exposure.
- pubsub usage: direct, moderate exposure (`pkg/crier/reporters/pubsub`, `pkg/pubsub/subscriber/*`, integration fake) — real reporting/trigger path but not auth/crypto-sensitive itself.
- secretmanager usage: direct, single call site `cmd/webhook-server/secretmanager/secretmanager.go` — sensitive (webhook secret fetching) but version delta (1.14.5→1.16.0, both mature 2025 releases) gives no cause for concern.
- storage usage: direct, moderate/broad (`pkg/io/{iterator,option,opener}.go`, gerrit/tide history tests) — generic blob abstraction, not credential-sensitive itself.
- google.golang.org/api: direct but narrow surface here (7 files); changelog (458 commits) dominated by unrelated generated services, not a useful signal at this granularity.
- Indirect-only churn (cloud.google.com/go core, auth, iam, longrunning, otel, gax-go) not deep-dived — standard lockstep bump, unremarkable.

## Open questions

None.

## Dependency followups

No improvement opportunities identified (mined 2026-10-27 against PR #866, head `248712a806947b17c5b0b63425f18f561e6a1f87`, merged as `fc981531d74bb22d73916199e35e944b8c09fc38` on 2026-08-24T21:21:02Z).

- `cloud.google.com/go/cloudbuild` 1.25.0 → 1.33.0 (direct; `pkg/googlecloudbuild/client/{client,fake/fake}.go`) — releases v1.26.0–v1.33.0 are "update sources/API and regenerate" plus LRO o11y tracing regeneration; no deprecation or replacement for `cloudbuild.NewClient`, `GetBuild`, `ListBuildsByTag`, `CreateBuild`, `CancelBuild`.
- `cloud.google.com/go/auth` 0.18.2 → 0.20.0 (direct; only `credentials.DetectDefault` in `pkg/io/credentials.go:32`) — v0.19.0/v0.20.0 add OTel gRPC/HTTP wrappers and actionable-error logging inside auth transports we do not use; `credentials.DetectOptions` unchanged.
- `google.golang.org/api` 0.274.0 → 0.287.1 (direct; only `option`, `iterator`, `googleapi`) — v0.275.0–v0.287.1 are auto-regenerated discovery clients plus a TODO removal; the core packages we import are untouched.
- `google.golang.org/genproto/googleapis/{api,rpc}` 2026-05-26 → 2026-06-30 pseudoversions — generated protos only, no changelog surface.
- `github.com/googleapis/gax-go/v2` 2.21.0 → 2.23.0 (indirect; not imported) — adds `http.response.status_code` to `TransportTelemetryData`, no call site of ours.
- `cloud.google.com/go/iam` 1.7.0 → 1.11.0 and `cloud.google.com/go/longrunning` 0.8.0 → 1.2.0 (indirect; not imported) — regen and o11y instrumentation (`longrunning` v1.2.0 adds poll-attempt/sleep spans behind experimental gating).
- `github.com/googleapis/enterprise-certificate-proxy` 0.3.14 → 0.3.17 (indirect; not imported) — internal mTLS filtering simplification in the ECP proxy.
- `go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc` 0.65.0 → 0.67.0 (indirect; not imported) — new `WithSpanKind` option and an `elapsedTime` metric fix; Prow builds no otelgrpc handlers directly.

Reason: no in-range changelog deprecates or replaces an API Prow calls, and every new feature lands in transports or packages Prow does not import directly.
