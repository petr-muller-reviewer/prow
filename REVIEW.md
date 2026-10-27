---
pr: kubernetes-sigs/prow#1033
title: "chore(deps): bump cloud.google.com/go/storage from 1.67.1 to 1.70.0"
head_sha: 56c4abbe5b4a4c3edf71a4405982c5e3007ad702
base: main
reviewed_at: 2026-10-27T23:37:21Z
verdict: needs-discussion
---

# Review

## Verdict

Wait for more soak before taking the storage release. OSV reports no advisories and the changelog changes most relevant to storage are for gRPC, which Prow does not select, but v1.70.0 is only about six days old and has no security fix that calls for taking it early.

## What this PR does

- Updates the direct `cloud.google.com/go/storage` dependency from v1.67.1 to v1.70.0.
- Resolves associated module versions: `cloud.google.com/go/auth` moves from an October 1 pseudo-version to v0.24.1, and `google.golang.org/api` moves from v0.305.0 to v0.307.0.
- Updates the indirect `google.golang.org/genproto/googleapis/rpc` pseudo-version from October 5 to October 19.
- Changes only `go.mod` and `go.sum`; no project source code changes.

## Findings

### [question] Allow the storage release more soak time
- where: `go.mod:10`
- concern: v1.70.0 was released on October 22, about 5 days 19 hours before this review. There are no known advisories or security fixes motivating an early update; consider waiting until it has about two weeks of soak.
- excerpt: |
    cloud.google.com/go/storage v1.70.0

## Checked

- `cloud.google.com/go/storage` is a direct dependency imported by three production files in `pkg/io` and four test files. The `pkg/io` abstraction is used across Prow for GCS-backed reads, writes, listings, metadata, and signed URLs.
- Prow creates the storage client with `storage.NewClient`, which uses the HTTP transport. No project call to `NewGRPCClient` or storage gRPC options was found. The storage release's principal retry and timeout changes target gRPC reads and writes, so those changes are not exercised by this code path.
- The storage changelog also includes a tracing attribute; it contains no storage security fix between v1.67.1 and v1.70.0.
- OSV returned no advisories for the old or new storage version. It also returned none for the old/new auth, Google API, or genproto RPC versions changed alongside it.
- `cloud.google.com/go/auth` v0.24.1 fixes empty TLS certificate handling after a secureconnect helper failure. `google.golang.org/api` v0.306.0 fixes deferring chunk context cancellation until the response body closes; v0.306 and v0.307 also regenerate discovery clients.
- `govulncheck -format json ./...` was attempted at base and PR head. Both runs failed while loading `encoding/json/jsontext` because the Go 1.26.10 environment excludes all files in that package by build constraints, so reachability could not be compared.

## Open questions

- Is there an urgent reason to take storage v1.70.0 before it has about two weeks of soak?

## Dependency followups

No actionable follow-up opportunities identified. Examined `cloud.google.com/go/storage` v1.67.1→v1.70.0, `cloud.google.com/go/auth` v0.24.1-0.20261001053825-dbc26066f70a→v0.24.1, `google.golang.org/api` v0.305.0→v0.307.0, `google.golang.org/genproto/googleapis/rpc` v0.0.0-20261005182115-fad411399dd8→v0.0.0-20261019180522-ce908d29299b, and `go.opentelemetry.io/otel/exporters/stdout/stdoutmetric` v1.44.0→v1.45.0 from `go.sum`. The storage improvements are mainly for gRPC, which Prow does not call; the auth and API changes are automatic dependency fixes, and Prow does not import the updated generated API clients or the stdout metric exporter.
