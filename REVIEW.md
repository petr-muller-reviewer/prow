---
pr: kubernetes-sigs/prow#973
title: "chore(deps): bump go.opentelemetry.io/otel/exporters/otlp/otlptrace from 1.44.0 to 1.45.0"
head_sha: 25dca66b6ceb5fe2da1797fe24ab7067a37d626d
base: main
reviewed_at: 2026-09-27T14:00:05Z
verdict: approve
---

# Review

## Verdict

Approve. This is a dependency-only change with adequately aged tagged releases, no Prow import surface for either updated module, and a security fix in `otlptrace` that removes exporter endpoint configuration from verbose internal OpenTelemetry logs. No new or reachable vulnerability findings were introduced.

## What this PR does

- Updates indirect `go.opentelemetry.io/otel/exporters/otlp/otlptrace` from `v1.44.0` to `v1.45.0`.
- Updates its selected indirect OTLP protobuf dependency, `go.opentelemetry.io/proto/otlp`, from `v1.10.0` to `v1.11.0`.
- Refreshes the corresponding checksums in `go.sum`.

## Findings

None.

## Resolved

None.

## Checked

- Classified the diff against the PR commit parent: only `go.mod` and `go.sum` change.
- Confirmed both changed modules are indirect and Prow has no Go imports of `go.opentelemetry.io` packages; their shortest dependency paths pass through Tekton/Knative observability.
- Verified release provenance and age: `otlptrace v1.45.0` was tagged 2026-08-03 (54 days old), and `proto/otlp v1.11.0` on 2026-07-22 (66 days old).
- Queried OSV: `GHSA-8wmf-6v46-5gfg` / `CVE-2026-81870` affects `otlptrace v1.44.0` and is fixed in `v1.45.0`; neither new version has an OSV advisory.
- Ran `govulncheck -format json ./...` on both the parent and PR head. Both completed with the same unrelated findings; neither reports this OpenTelemetry advisory as reachable.
- Reviewed the upstream module delta: it prevents recursive logging of OTLP exporter client configuration and adds OTLP MAP attribute serialization. Prow does not directly exercise either surface.
- Ran `git diff --check` successfully.

## Open questions

None.

## Dependency followups

No opportunities identified; Prow imports neither changed module.

- Examined indirect `go.opentelemetry.io/otel/exporters/otlp/otlptrace` `v1.44.0` → `v1.45.0`: [the release](https://github.com/open-telemetry/opentelemetry-go/releases/tag/v1.45.0) adds MAP attribute serialization and removes endpoint configuration from internal logs, but Prow has no call sites to change. Its `WithEndpointURL` behavior change is in the separately versioned `otlptracehttp` module, which this PR leaves at `v1.44.0`.
- Examined indirect `go.opentelemetry.io/proto/otlp` `v1.10.0` → `v1.11.0`: [the release](https://github.com/open-telemetry/opentelemetry-proto-go/releases/tag/v1.11.0) updates generated OTLP protocol types, which Prow does not import.
