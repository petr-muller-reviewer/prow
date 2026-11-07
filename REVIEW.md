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
