---
pr: kubernetes-sigs/prow#991
title: "chore(deps): bump google.golang.org/api from 0.297.0 to 0.299.0"
head_sha: 2a8d7179365942431aba01cdc52aae2e081826cf
base: main
reviewed_at: 2026-10-05T23:15:10Z
verdict: approve
---

# Review

## Verdict

Approve for Prow's current configuration. The bump is dep-only, and the API/auth changes fit Prow's existing Google Cloud usage. The gRPC update brings in OSV advisory GO-2026-6443, but its panic condition requires xDS routing; Gangway creates a plain gRPC server and the scoped head scan found no call to the xDS route handler. Reassess if Prow enables xDS routing.

## What this PR does

- Raises `google.golang.org/api` from `v0.297.0` to `v0.299.0`.
- Updates direct Google auth and gRPC requirements to the versions required by the new API module.
- Refreshes five indirect Google transport, retry, and protocol modules; there are no Prow source changes.

## Findings

None for Prow's current configuration. The conditional gRPC advisory and reachability evidence are recorded under Checked.

## Checked

- **Classification:** dep-only. The merge-base diff contains only `go.mod` and `go.sum`.
- **Versions, freshness, and OSV:** queried the Go module proxy and OSV for old and new versions. All old versions and all new versions below have no OSV advisories; `grpc v1.84.0` is the exception.

  | Module | Change | Kind | New version date and age at review | OSV |
  |---|---|---|---|---|
  | `google.golang.org/api` | `v0.297.0 → v0.299.0` | Direct | 2026-09-21, 14 days | None |
  | `cloud.google.com/go/auth` | `v0.23.2 → v0.23.3` | Direct | 2026-09-17, 18 days | None |
  | `google.golang.org/grpc` | `v1.83.2 → v1.84.0` | Direct | 2026-09-17, 18 days | GO-2026-6443 on new version |
  | `cloud.google.com/go/compute/metadata` | `v0.9.0 → v0.9.1` | Indirect | 2026-09-17, 18 days | None |
  | `github.com/google/s2a-go` | `v0.1.9 → v0.1.10` | Indirect | GitHub published 2026-09-17, 18 days; proxy reports 2025-09-29 commit time | None |
  | `github.com/googleapis/enterprise-certificate-proxy` | `v0.3.21 → v0.3.22` | Indirect | 2026-09-09, 26 days | None |
  | `github.com/googleapis/gax-go/v2` | `v2.24.0 → v2.24.1` | Indirect | 2026-09-03, 32 days | None |
  | `google.golang.org/genproto/googleapis/rpc` | `v0.0.0-20260825221802-da73d73af1c5 → v0.0.0-20260921155816-b14227669459` | Indirect pseudo-version | 2026-09-21, 14 days | None |

- **API usage and exposure:** Prow directly imports `google.golang.org/api` in 9 Go files (6 production files in `pkg/io`, `pkg/googlecloudbuild/client`, and `cmd/webhook-server/secretmanager`; 3 tests). It provides client options, pagination, and API error types. Prow uses it on GCS, Cloud Build, and Secret Manager paths. The shared credentials helper loads operator-provided Google credentials in [pkg/io/credentials.go](/workspace/pkg/io/credentials.go:31), so this is a meaningful auth and cloud API surface.
- **API changelog:** [v0.298.0](https://github.com/googleapis/google-api-go-client/releases/tag/v0.298.0) and [v0.299.0](https://github.com/googleapis/google-api-go-client/releases/tag/v0.299.0) regenerate discovery clients. The generated Storage API adds `ExtendedDataTypeUrl`. The API module also now cancels the completed context in its resumable-upload helper. Prow's GCS opener uses the HTTP Storage client and writes objects, so the cleanup change can apply to Prow uploads; it does not alter upload results or API signatures.
- **Auth and indirect changelogs:** [Cloud Auth v0.23.3](https://github.com/googleapis/google-cloud-go/releases/tag/auth/v0.23.3) handles secureconnect helper failures more gracefully. `compute/metadata v0.9.1` adds a timeout to the initial metadata subscriber request, relevant to Google credential discovery. S2A adjusts TLS identity handling; the certificate proxy release updates its version; `gax v2.24.1` changes ProtoJSON stream selection for Go 1.27+; genproto refreshes generated definitions. No material behavior change was found in Prow's direct call sites for these indirect updates.
- **gRPC changelog and vulnerability:** [gRPC v1.84.0](https://github.com/grpc/grpc-go/releases/tag/v1.84.0) includes client fixes and credential/metadata validation, plus STS redirect protection against token leakage. OSV [GO-2026-6443](https://osv.dev/vulnerability/GO-2026-6443) is absent from `v1.83.2` and present in `v1.84.0`; it describes a server panic when xDS routing handles requests missing both authority and Host headers. OSV lists a development commit as fixed; `v1.84.0` was the latest stable gRPC release at review time.
- **Reachability:** full `govulncheck ./...` completed at the base and did not report GO-2026-6443. The full head scan was killed by OOM (exit 137), so the relevant package was checked separately. Scoped `govulncheck ./cmd/gangway` completed at base without the advisory; at head it found `grpc/internal/transport.HandleStreams` reachable through Gangway's server. The xDS server package was import-only and there was no call trace to its route handler. Gangway constructs a plain `grpc.NewServer` at [cmd/gangway/main.go](/workspace/cmd/gangway/main.go:198), without xDS routing configuration. The advisory's required route path is therefore not reached by the reviewed configuration.
- **Dependency relationship:** `google.golang.org/api v0.299.0` itself requires `cloud.google.com/go/auth v0.23.3` and `google.golang.org/grpc v1.84.0`. gRPC is used in 11 Go files, including Gangway's server and Prow's gRPC clients; this is a network-sensitive dependency, but the advisory's xDS-specific condition is not enabled here.

## Open questions

None.

## Dependency followups

### [new-feature] Add actionable Google client failure logging
- module: `github.com/googleapis/gax-go/v2` `v2.24.0 → v2.26.2`, unlocked by open PR #991, “chore(deps): bump google.golang.org/api from 0.297.0 to 0.300.0”
- where: `pkg/googlecloudbuild/client/client.go:92-103,122-197`; `cmd/webhook-server/secretmanager/secretmanager.go:48-55,58-173`
- necessity: could — optional structured diagnostics for terminal Google API failures.
- changelog: GAX v2.26.0 adds “ClientLogging configuration,” “WithClientLogging CallOption,” and “actionable error logging in Invoke.” The warning includes RPC method and error/retry metadata on terminal failure.
- outcome: accepted
- handoff prompt:

  ```text
  In kubernetes-sigs/prow, now that open PR #991, “chore(deps): bump google.golang.org/api from 0.297.0 to 0.300.0,” upgrades github.com/googleapis/gax-go/v2 from v2.24.0 to v2.26.2, evaluate and add GAX actionable failure logging to the Google client wrappers in pkg/googlecloudbuild/client/client.go and cmd/webhook-server/secretmanager/secretmanager.go. GAX v2.26.0 added ClientLogging configuration, a WithClientLogging CallOption, and actionable error logging in Invoke. Confirm the generated Cloud Build and Secret Manager clients accept and execute this option; if so, create one logging configuration per client and route warning events through Prow’s structured logger. Log only method, retry, status, and error metadata; never log request payloads or secret values. Add focused tests for failure-only logging and sensitive-data exclusion. If these clients cannot support the option cleanly, document why and leave source unchanged. Keep the work limited to these wrappers and GAX logging; do not change dependency versions or expand into general observability work.
  ```
