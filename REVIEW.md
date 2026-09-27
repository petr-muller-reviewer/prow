---
pr: kubernetes-sigs/prow#974
title: "chore(deps): bump go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc from 1.44.0 to 1.45.0"
head_sha: 164b488451f2bffb30cb8d50bb24e951eaab4bd2
base: main
reviewed_at: 2026-09-27T11:53:24Z
verdict: approve
---

# Review

## Verdict

Approve. This is a sufficiently-soaked, tagged, indirect dependency bump that removes a known endpoint-information disclosure in the OTLP gRPC trace exporter. The module is present only through the pipeline controller's Tekton/Knative tracing stack; Prow does not import it directly, and no project code changes accompany the update.

## What this PR does

- Updates the indirect OTLP gRPC trace exporter requirement from `v1.44.0` to `v1.45.0` in `go.mod`.
- Regenerates the corresponding module checksums in `go.sum`.
- Takes the upstream fix that stops the exporter from including its configured endpoint in internal log output.
- Does not modify Prow source, tests, configuration, or generated project artifacts.

## Findings

No findings.

## Resolved

None.

## Checked

- `go.mod:229` marks the bumped module as indirect; the PR changes only `go.mod` and `go.sum`.
- `go mod why -m go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc` reaches it through `cmd/pipeline`, Tekton, and Knative observability tracing; repository source has no direct importer.
- The tagged `v1.45.0` release was cut on 2026-08-03, 55 days before review.
- OSV reports `GHSA-8wmf-6v46-5gfg` / `CVE-2026-81870` for `v1.44.0` and none for `v1.45.0`; `govulncheck` was unavailable in this environment.
- Upstream's relevant gRPC exporter change removes the endpoint field from `MarshalLog`, addressing the advisory without changing Prow-owned code.

## Open questions

None.

## Dependency followups

No opportunities identified for `go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc` `v1.44.0 → v1.45.0`; no other module versions changed. Prow has no direct OpenTelemetry imports or calls to the affected APIs, and the exporter changes are internal fixes and generated-code updates rather than features requiring Prow changes.
