---
pr: kubernetes-sigs/prow#833
title: "chore(deps): bump google.golang.org/grpc from 1.79.3 to 1.82.1 in /hack/tools"
head_sha: b746fdd01caa2b9d78d484da5b417777d787f6c5
base: main
reviewed_at: 2026-08-18T15:13:19Z
verdict: approve
---

## What this PR does

- Dependabot bump of `google.golang.org/grpc` from v1.79.3 to v1.82.1 in the `hack/tools` submodule only.
- `hack/tools/go.mod`/`go.sum` changed (12/36 lines); no project source touched.
- Side-effect transitive bumps in the same module: `go.opentelemetry.io/otel{,/metric,/trace}` v1.42.0→v1.43.0, `google.golang.org/genproto/googleapis/{api,rpc}` pseudo-versions 2026-03-16→2026-04-14.
- `hack/tools` is a build-tooling-only Go module (see `hack/tools/tools.go`) pinning binary deps for golangci-lint, k8s code-generator, controller-gen, protoc-gen-go, gotestsum, misspell, ko. grpc is indirect there, pulled in transitively by one of those tools, not imported by any prow source.
- Root `go.mod` (actual prow binaries) has its own direct grpc dependency, unaffected by this PR, still pinned at v1.79.3.

## Dependency analysis

- Freshness: v1.82.1 released 2026-07-15 (real tag, not pseudo-version) — ~34 days old at review time. Past soak window, fine.
- Usage: indirect, zero import surface in prow's own code; confined to `hack/tools` build tooling.
- Changelog 1.79.3→1.82.1: xDS/RBAC bug fixes, balancer registry case-sensitivity change, and a security fix in 1.82.1 (server-side HTTP/2 frame-flood DoS mitigation, xDS/RBAC ACL matcher fail-open fixes). None of this is reachable here since this copy of grpc only runs inside dev-time build/lint/codegen tooling, not any prow server binary.

## Findings

(none)

## Checked
- Diff scope confined to `hack/tools/go.mod` and `hack/tools/go.sum` — no project code changes.
- Root module's grpc dependency (v1.79.3, direct) is untouched and unaffected by this bump.
- grpc import surface in prow source: none (indirect-only, dev tooling module).
- New grpc release provenance: real tagged release on grpc-go GitHub, not a pseudo-version.

## Open questions
(none)

## Dependency followups

Scope examined: PR #833 (merged `e95f0a79`) bumping `google.golang.org/grpc` v1.79.3→v1.82.1 in `hack/tools`, with its transitive `go.opentelemetry.io/otel{,/metric,/trace}` v1.42.0→v1.43.0 and `google.golang.org/genproto/googleapis/{api,rpc}` pseudo-version move 2026-03-16→2026-04-14.

Per-module outcome:
- `google.golang.org/grpc` v1.79.3→v1.82.1 — one opportunity accepted (below). Deprecations in the range (`WithCompressor` and its `gzip` compressor) and new features (xDS SNI/SAN, otel custom per-call labels, `mem.Buffer.Slice`, `WithSharedWriteBuffer`) have no call site in prow, so nothing else to adopt.
- `go.opentelemetry.io/otel{,/metric,/trace}` v1.42.0→v1.43.0 — root module already pins v1.43.0 (`go.mod:169-171`); nothing new for us.
- `google.golang.org/genproto/googleapis/{api,rpc}` — generated stubs, no changelog; root's older pins for its own module are routine Dependabot territory, not a changelog-driven followup.

Tally: 1 accepted, 0 skipped.

### [api-upgrade] Root module's grpc is still v1.79.3, missing the 1.82.1 frame-flood mitigation
- module: `google.golang.org/grpc` v1.79.3→v1.82.1 (this PR bumps only the `hack/tools` module; root module untouched)
- where: root pin `go.mod:74`; server `cmd/gangway/main.go:198` (`grpc.NewServer`); clients `pkg/resultstore/client.go:45`, `pkg/gangway/client/google/google.go:96`, `pkg/gangway/client/google/google.go:181`, `test/integration/internal/fakepubsub/fakepubsub.go:64`
- changelog (v1.82.1, Security): "server: Stop reading from the connection when flooded by HTTP/2 frames. The default value for this limit is 100 frames, excluding DATA and HEADERS, and may be changed by setting environment variable `GRPC_GO_EXPERIMENTAL_CONTROL_BUFFER_THROTTLE_LIMIT`."
- necessity: should — defense in depth for gangway's gRPC endpoint; not a regression introduced by this PR and not a merge blocker.
- scope: bump root `go.mod` to v1.82.1 plus `go mod tidy`; no call-site changes expected, our usage (`NewClient`/`NewServer`, `WithTransportCredentials`, `WithPerRPCCredentials`) is unaffected by the 1.80–1.82 behavior changes.

#### Handoff prompt

```text
In kubernetes-sigs/prow, PR #833 ("chore(deps): bump google.golang.org/grpc from 1.79.3 to 1.82.1 in /hack/tools", merged as e95f0a79bf3e4f75e8bb3d2ef4b8c98372281ff6) bumped grpc only in the hack/tools module. The root module still pins google.golang.org/grpc v1.79.3 (go.mod:74), which is what prow's runtime binaries build against.

Task: bump google.golang.org/grpc to v1.82.1 in the root go.mod and run `go mod tidy` afterwards, so prow's binaries - notably the gangway gRPC server built with grpc.NewServer at cmd/gangway/main.go:198 - pick up the v1.82.1 server mitigation: "server: Stop reading from the connection when flooded by HTTP/2 frames" (default limit 100 frames, excluding DATA and HEADERS). Review the resulting go.sum/transitive churn; it should be small, since the root module already pins go.opentelemetry.io/otel v1.43.0.

Call sites that must keep compiling unchanged: cmd/gangway/main.go:198 (NewServer), pkg/resultstore/client.go:45 (NewClient), pkg/gangway/client/google/google.go:96 and :181 (NewClient), test/integration/internal/fakepubsub/fakepubsub.go:64 (NewClient). They use stable APIs (WithTransportCredentials, WithPerRPCCredentials) that the 1.80-1.82 behavior changes do not affect, so no code edits should be necessary.

Acceptance criteria:
- root go.mod requires google.golang.org/grpc v1.82.1 and `go mod tidy` leaves no further diff.
- `go build ./...` passes.
- `go test ./pkg/gangway/... ./pkg/resultstore/...` passes, or any failure is shown to be pre-existing and unrelated to the bump.

Scope guard: dependency bump only. Do not change gangway server options, interceptors, or deployment config, and do not touch the hack/tools module (already on v1.82.1). Do not refactor grpc call sites to newer APIs.
```
