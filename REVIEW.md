---
pr: kubernetes-sigs/prow#713
title: "Bump Kubernetes dependencies to v0.33.11"
head_sha: 370e42dc157d5cf743ef9e6c0139cdd572d96092
base: main
reviewed_at: 2026-09-28T17:39:56Z
verdict: approve
---

# Review

## Verdict

Approve — no defect found in the dependency bump or accompanying code changes.

The Kubernetes releases had about 25–27 days of soak time when this PR merged on May 11, 2026. The direct dependency changes affect core Kubernetes clients and controllers, but the API adaptations match the new interfaces, OSV reports no advisory for either endpoint version of the bumped modules, and the scoped vulnerability scan found no new advisory IDs. This is a retrospective verdict: Kubernetes 1.33 reached end of life on June 28, 2026, so v0.33.11 is no longer a suitable current upgrade target.

## What this PR does

- Updates `k8s.io/api` v0.32.8 → v0.33.11, `k8s.io/apimachinery` v0.32.9 → v0.33.11, and `k8s.io/client-go` v0.32.8 → v0.33.11.
- Updates `sigs.k8s.io/controller-runtime` v0.20.1 → v0.21.0 and `github.com/prometheus/client_golang` v1.20.5 → v1.22.0, plus four indirect modules and corresponding sums.
- Passes the report context to client-go's `SearchWithContext` in `pkg/crier/reporters/gcs/kubernetes/reporter.go:105` and adapts the test fake.
- Handles `AddEventHandlerWithOptions` in the Plank test at `pkg/plank/reconciler_test.go:284` and regenerates the ProwJob CRD and plugin config example.

## Findings

None.

## Checked

- Classified the PR as dependency + code; reviewed the reporter, test, CRD, and generated config changes against the PR's stated API and test fixes. No standards or behavior finding.
- Kubernetes v0.33.11 modules were published April 14–16, 2026; they are about 165–167 days old as of this review. The other direct releases are older. Source: [Kubernetes v1.33 changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.33.md).
- Import surface in production Go files: `k8s.io/api` 162 files/113 packages; `k8s.io/apimachinery` 157/109; `k8s.io/client-go` 59/35; `sigs.k8s.io/controller-runtime` 30/27; `github.com/prometheus/client_golang` 35/28. Kubernetes APIs, serialization, network clients, and Plank reconciliation are core and sensitive paths; metrics exposure is broad but less sensitive.
- The indirect changes are `k8s.io/apiextensions-apiserver` v0.32.8 → v0.33.11, `k8s.io/kube-openapi` v0.0.0-20241212222426-2c72e554b1e7 → v0.0.0-20250318190949-c8a335a9a2ff, `sigs.k8s.io/structured-merge-diff/v4` v4.5.0 → v4.6.0, and `github.com/gorilla/websocket` v1.5.3 → v1.5.4-0.20250319132907-e064f32e3674. None is imported directly in Prow Go files. The kube-openapi and websocket targets are untagged pseudo-versions; both were over a year old at review time.
- [Controller-runtime v0.21.0](https://github.com/kubernetes-sigs/controller-runtime/releases/tag/v0.21.0) changes default client rate limiting and informer registration. The registration change reaches Plank's test wrapper; the new method mirrors the existing signal handling.
- [Prometheus client v1.22.0](https://github.com/prometheus/client_golang/releases/tag/v1.22.0) makes experimental zstd scrape support opt-in. Prow uses `promhttp.HandlerFor` in `pkg/metrics/metrics.go:61` without a zstd import; scrapes requesting zstd may receive an uncompressed response.
- OSV queries for old and new versions of all nine bumped modules returned no advisories. Scoped `govulncheck` runs over `./pkg/plank/... ./pkg/crier/... ./cmd/...` at the PR base and head found no findings in the bumped modules and no newly introduced advisory IDs. An imported-only finding for removed indirect dependency `github.com/klauspost/compress` (GO-2026-5841) disappeared; unrelated pre-existing findings remained.
- [Kubernetes release lifecycle](https://kubernetes.io/releases/) lists 1.33 as end of life on June 28, 2026. This affects any present-day upgrade decision, not the historical merge verdict.

## Open questions

None.

## Dependency followups

No outstanding improvement opportunities identified. Examined the direct bumps `k8s.io/api` v0.32.8 → v0.33.11, `k8s.io/apimachinery` v0.32.9 → v0.33.11, `k8s.io/client-go` v0.32.8 → v0.33.11, `sigs.k8s.io/controller-runtime` v0.20.1 → v0.21.0, and `github.com/prometheus/client_golang` v1.20.5 → v1.22.0 against Prow's call sites; also checked the indirect changes to `k8s.io/apiextensions-apiserver` v0.32.8 → v0.33.11, `k8s.io/kube-openapi` v0.0.0-20241212222426-2c72e554b1e7 → v0.0.0-20250318190949-c8a335a9a2ff, `sigs.k8s.io/structured-merge-diff/v4` v4.5.0 → v4.6.0, and `github.com/gorilla/websocket` v1.5.3 → v1.5.4-0.20250319132907-e064f32e3674. The indirect modules have no direct Prow imports; the other new APIs and deprecations do not match an actionable current call site.

The client-go 0.33 `ListWithContextFunc` and `WatchFuncWithContext` APIs were relevant to Prow's generated ProwJob and Pipeline informers, which used `context.TODO()` at this PR head. The current `upstream/main` already uses both context-aware functions in those informers and has updated `k8s.io/code-generator`, so this potential followup is complete and needs no handoff. `Result.Requeue` and the deprecated Kubernetes Endpoints and `resource.k8s.io/v1beta1` APIs have no Prow call sites. Prometheus's new `CollectorFunc` does not fit the exporter's custom collector, whose descriptors have dynamic label sets.
