---
pr: kubernetes-sigs/prow#1090
title: "chore(deps): bump google.golang.org/grpc from 1.84.0 to 1.85.0"
head_sha: e9f9ae3981bb9eac95483440d9068596af3c077f
base: main
reviewed_at: 2026-11-23T12:24:55Z
verdict: approve
---

# Review

## Verdict

Safe to bump now — approve.

Dep-only bump of `google.golang.org/grpc` `v1.84.0 → v1.85.0` plus the indirect
`envoyproxy/go-control-plane/envoy` pin that grpc 1.85.0 requires. The release is 27 days
old and properly tagged, OSV has no advisories at either version, and none of the changed
APIs are used by prow. The server fixes in 1.85 land directly in `gangway`'s gRPC server
(stream-ID handling, `HEADERS`+`EndStream` hang, missing `:authority` rejection, forceful
`Stop()`), so this is a net hardening rather than a risk.

## What this PR does

- Bumps the direct dependency `google.golang.org/grpc` from `v1.84.0` to `v1.85.0` (`go.mod:74`).
- Pulls the indirect `github.com/envoyproxy/go-control-plane/envoy` pin from the `v1.39.0` tag
  to pseudo-version `v1.39.1-0.20260819172001-e6e3fd93e4be`, as required by grpc 1.85.0's own
  `go.mod`; nothing in prow selects this version.
- Touches only `go.mod` and `go.sum` (6 insertions, 6 deletions); no vendor tree exists and no
  project code or generated artifacts change.
- Brings in grpc server hardening (`Stop()` during `GracefulStop()`, `HEADERS`+`EndStream`,
  RFC 7540 stream-ID increment, requests without `:authority`/`Host`), transport control-buffer
  throttle tuning, and a retry `:authority` fix for xDS load balancers (not used here).

## Findings

None. Dependency-only bump with no advisories fixed or open, no API removals affecting our import
surface, and no project code to review.

## Checked

- Freshness: `v1.85.0` tagged `2026-10-27T09:27:04Z` (`refs/tags/v1.85.0`, commit `c9ab42c`),
  27 days before review — enough soak time; real tag, not a pseudo-version.
- Directness: direct dependency; `go mod why -m google.golang.org/grpc` →
  `sigs.k8s.io/prow/cmd/gangway` → `google.golang.org/grpc`.
- Import surface: 11 non-vendored files across ~8 packages — `cmd/gangway/main.go:200` (gRPC
  server, health, reflection), `pkg/gangway/gangway_grpc.pb.go`, `pkg/gangway/gangway.go`,
  `pkg/gangway/client/google/google.go:96,181` (TLS and insecure clients),
  `pkg/resultstore/client.go:45-49` (OAuth per-RPC credentials), `pkg/resultstore/writer/writer.go`,
  `pkg/crier/reporters/pubsub/reporter.go` (`codes`/`status`), `pkg/config/gangway.go:25`, plus
  two test helpers.
- Sensitive paths: yes — network-facing `gangway` gRPC server plus TLS/OAuth credential
  handling on clients; changelog changes land in exactly those server-side code paths and are
  fixes/hardening.
- OSV: no advisories for `google.golang.org/grpc` at `v1.84.0` or `v1.85.0`, and none for the
  envoy pseudo-version.
- `govulncheck` module scan at merge-base and PR head: no grpc findings on either side. Both
  sides report the same unrelated, untouched-module advisories: `GO-2026-5932`
  (`golang.org/x/crypto v0.58.0`) and `GO-2023-1901` (`github.com/tektoncd/pipeline v1.16.0`);
  neither module changes in this PR. Module scan had to run from `cmd/gangway` because the repo
  root has no Go files and `-scan=module` rejects package patterns.
- Symbol scan at head was attempted and killed (exit 137, memory limit) after ~2.5 min; partial
  output contained no grpc findings — only `GO-2023-1901` at package level with no `function`
  (imported/required, not called) and module-level `GO-2026-5932`. Neither is bump-related.
- Changelog: 52 commits, 13 release-note items; new APIs are additive (`stats.OutHeader` client
  authority, `metadata.ValueFromOutgoingContext`); no security advisories and no deprecations or
  removals affecting our usage (`WithInitialWindowSize` is not used anywhere in the repo).
- Build: `go build ./cmd/gangway/... ./pkg/gangway/... ./pkg/resultstore/... ./pkg/crier/reporters/pubsub/... ./pkg/config/...` exits 0.
- Envoy indirect: pseudo-version from 2026-08-19 replaces tag `v1.39.0`; reachable only via
  `pkg/io` → `cloud.google.com/go/storage` → `grpc/stats/opentelemetry.test` →
  `envoy/config/cluster/v3`, i.e. test-time only and not linked into prow binaries.

## Open questions

None. The envoy pseudo-version is forced by grpc's own `go.mod`, so there is nothing actionable
to raise on this PR.

## Dependency followups

No improvement opportunities identified from this bump.

- `google.golang.org/grpc` `v1.84.0 → v1.85.0` (direct): the only deprecation in the range is
  `grpc.WithInitialWindowSize` / `grpc.InitialWindowSize`, replaced by
  `WithInitialStreamWindowSize` / `InitialStreamWindowSize`; prow sets no window-size options at
  all (no non-vendor `WindowSize` hits). The new APIs have no call site here:
  `metadata.ValueFromOutgoingContext` (our `metadata` use only appends to or creates outgoing
  context and never reads it back), `stats.OutHeader` client authority (no `grpc/stats` handler
  registered), `balancer/autosharding`, `credentials/google` ID-token credentials (not imported;
  `pkg/resultstore/client.go` uses the non-deprecated `oauth.NewApplicationDefault` from the
  unchanged `credentials/oauth` package), and the xDS `GrpcService`/server filter APIs.
- `github.com/envoyproxy/go-control-plane/envoy` `v1.39.0 → v1.39.1-0.20260819172001-e6e3fd93e4be`
  (indirect): never imported by prow; reachable only through `cloud.google.com/go/storage` →
  `grpc/stats/opentelemetry.test` → `envoy/config/cluster/v3`, so there is no code of ours to change.
