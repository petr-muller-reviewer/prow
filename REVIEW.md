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
