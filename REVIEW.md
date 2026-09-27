---
pr: kubernetes-sigs/prow#975
title: "chore(deps): bump go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp from 1.44.0 to 1.45.0"
head_sha: e1c61e181176e5334a369982b805d79361374b21
base: main
reviewed_at: 2026-09-27T12:59:24Z
verdict: approve
---

# Review

## Verdict

Approve. This is a sufficiently soaked, tagged dependency-only update that removes the known endpoint-logging disclosure advisory in the selected old version. The updated HTTP OTLP trace exporter is compiled through the Tekton tracing path, but Prow does not import it directly and the behavior changes do not alter Tekton's `WithEndpoint`/`WithURLPath` configuration.

## What this PR does

- Updates the indirect HTTP OTLP trace exporter from `v1.44.0` to `v1.45.0`.
- Refreshes the corresponding module checksums only; it makes no Prow source changes.
- Takes the fix for `GHSA-8wmf-6v46-5gfg` / `CVE-2026-81870`, which prevents exporter configuration logging from exposing endpoint URLs.
- Brings corrected `Retry-After` handling for OTLP/HTTP exporter retries.

## Findings

No findings.

## Resolved

None.

## Checked

- Classified the change as dependency-only: `go.mod` and `go.sum` are the only modified files.
- Confirmed the release is a tagged upstream `v1.45.0` release from 2026-08-03, 55 days before review.
- Queried OSV for both versions: the old version has `GHSA-8wmf-6v46-5gfg` / `CVE-2026-81870`; the new version has no reported advisory.
- Confirmed Prow has zero direct imports. The exporter is compiled through `cmd/pipeline` → Tekton → Knative tracing, and Tekton uses `WithEndpoint` plus `WithURLPath`, not the changed `WithEndpointURL` behavior.
- Ran `govulncheck -format json ./cmd/pipeline/... ./pkg/pipeline/...`; it found no reachable instance of the advisory fixed by this bump.
- Ran `git diff --check 01f306768904fcf3a371f83a4db040d502c9bf61..HEAD` successfully.

## Open questions

None.

## Dependency followups

No actionable opportunities identified.

- Examined `go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp` `v1.44.0` → `v1.45.0` (indirect), the only module version change in this PR.
- The [v1.45.0 release notes](https://github.com/open-telemetry/opentelemetry-go/releases/tag/v1.45.0) describe changed `WithEndpointURL` path handling and fixes to HTTP `Retry-After` handling and endpoint logging. Prow does not import this exporter or call its APIs in source; the exporter is reached through Tekton. There is no Prow-owned call site to migrate or simplify.
