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

## Dependency followups

Examined on 2026-10-20 for PR #890, merged as `ba214a6138015a5e1be604fb2a474778e6b6cc62`. No concrete opportunities identified; no items or handoff prompts.

The PR head is `84b2c62092ef3ab688ba282a736f9d165787adde`. Because the PR is already contained in `upstream/main`, its merge-base is the PR head itself; the module ranges below come from the PR commit's parent-to-head diff.

- `google.golang.org/grpc` (direct), `v1.82.1` → `v1.83.1`: examined the commit range and both intermediate [1.83.0 release notes](https://github.com/grpc/grpc-go/releases/tag/v1.83.0) and [1.83.1 release notes](https://github.com/grpc/grpc-go/releases/tag/v1.83.1). New xDS routing, proxy, metrics, and authentication features and the authz `OnPolicyUpdate` callback have no corresponding imports or call sites in Prow. No newly deprecated API used by Prow was identified. Gangway already uses `grpc.NewClient` (`pkg/gangway/client/google/google.go:96,181`), as does ResultStore (`pkg/resultstore/client.go:45`).
- The new `credentials/google.NewServiceAccountIdentityCredentials` is experimental and obtains GCP identity tokens from the metadata server; it cannot replace Gangway's explicit JSON-key, self-signed JWT contract at `pkg/gangway/client/google/google.go:76,121,146` without changing supported authentication environments. ResultStore uses OAuth access tokens, a different credential type. [Credential implementation and contract](https://github.com/grpc/grpc-go/blob/v1.83.1/credentials/google/gcp_service_account_identity_credentials.go).
- The 1.83.0 HTTP/2 control-frame flood protection applies automatically to Gangway's `grpc.NewServer` (`cmd/gangway/main.go:198`), and the 1.83.1 small-frame buffering improvement also requires no call-site changes. No existing override of `GRPC_GO_EXPERIMENTAL_CONTROL_BUFFER_THROTTLE_LIMIT` was found to remove or adjust.
- `cel.dev/expr` (indirect), `v0.25.1` → `v0.25.2`: [changes](https://github.com/google/cel-spec/compare/v0.25.1...v0.25.2) concern conformance tests, specification clarification, and publishing automation. Prow has no direct imports or API migration to perform.
- `github.com/GoogleCloudPlatform/opentelemetry-operations-go/detectors/gcp` (indirect), `v1.32.0` → `v1.33.0`: [detector changes](https://github.com/GoogleCloudPlatform/opentelemetry-operations-go/compare/detectors/gcp/v1.32.0...detectors/gcp/v1.33.0) add Cloud Run worker-pool detection and update dependencies. Prow does not import the detector or own a detector call site that could adopt the feature.
- `go.opentelemetry.io/contrib/detectors/gcp` (indirect), `v1.43.0` → `v1.44.0`: [module changes](https://github.com/open-telemetry/opentelemetry-go-contrib/compare/detectors/gcp/v1.43.0...detectors/gcp/v1.44.0) update semantic-convention imports and dependencies. Prow has no direct imports or redundant detector implementation to replace.
