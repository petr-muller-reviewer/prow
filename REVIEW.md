---
pr: kubernetes-sigs/prow#996
title: "chore(deps): bump the kubernetes group across 2 directories with 5 updates"
head_sha: 7c6d3c97f8a8d75e3326a8998ca3a7efc09d8f2d
base: main
reviewed_at: 2026-10-10T13:50:50Z
verdict: needs-discussion
---

# Review

## Verdict

Needs discussion: the bump has no PR-specific vulnerability or source-code regression, but `sigs.k8s.io/controller-runtime v0.25.2` is about 9–10 days old, and the branch retains reachable vulnerabilities in unchanged dependencies.

The Kubernetes module releases are older than two weeks and their tag changes do not alter package source. Decide whether the usual two-week soak applies to controller-runtime; track the reachable `x/net` and Tekton findings separately because this PR does not address them.

## What this PR does

- Updates Kubernetes module versions in the main module and tools module.
- Updates `sigs.k8s.io/controller-runtime` from `v0.25.1` to `v0.25.2`.
- Changes only `go.mod` and `go.sum` files; no project source or generated files changed.

## Findings

None specific to this dep-only PR.

## Resolved

None.

## Checked

- **Classification and manifests:** `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum` are the only changed files. Six unique modules are updated:
  - `k8s.io/api v0.37.0 → v0.37.1` — direct in the main module, indirect in tools; imported by 52 Go files (26 excluding tests), for Kubernetes API types.
  - `k8s.io/apimachinery v0.37.0 → v0.37.1` — direct in the main module, indirect in tools; imported by 281 files (160 excluding tests), including API machinery, serialization, and metadata paths.
  - `k8s.io/client-go v0.37.0 → v0.37.1` — direct; imported by 76 files (59 excluding tests) for Kubernetes clients, kubeconfig, authentication, watches, and informers.
  - `k8s.io/streaming v0.37.0 → v0.37.1` — indirect; no direct Prow imports found. `go mod why` traces it through `client-go/tools/remotecommand` from integration tests.
  - `sigs.k8s.io/controller-runtime v0.25.1 → v0.25.2` — direct; imported by 55 files (30 excluding tests), across controllers, caches, and clients.
  - `k8s.io/code-generator v0.37.0 → v0.37.1` — direct in `hack/tools`; used by the tools harness, not production code.
- **Freshness:** the `api`, `apimachinery`, `client-go`, and `code-generator` tags were cut September 23, 2026 (about 17 days old); `streaming` was cut August 9 (about 62 days old). `controller-runtime` was cut September 30 and published October 1 (about 9–10 days old). None is under five days old; controller-runtime is the only one still within the usual two-week soak window. All new versions are tagged releases, not pseudo-versions. Sources: [Kubernetes module proxy metadata](https://proxy.golang.org/k8s.io/api/@v/v0.37.1.info), [streaming module proxy metadata](https://proxy.golang.org/k8s.io/streaming/@v/v0.37.1.info), [controller-runtime module proxy metadata](https://proxy.golang.org/sigs.k8s.io/controller-runtime/@v/v0.25.2.info).
- **Kubernetes module changes:** tag comparisons for [api](https://github.com/kubernetes/api/compare/v0.37.0...v0.37.1), [apimachinery](https://github.com/kubernetes/apimachinery/compare/v0.37.0...v0.37.1), [client-go](https://github.com/kubernetes/client-go/compare/v0.37.0...v0.37.1), and [code-generator](https://github.com/kubernetes/code-generator/compare/v0.37.0...v0.37.1) show dependency metadata/tag alignment rather than package source changes; `streaming` has no source delta between the tags. Prow uses the Kubernetes API and client libraries heavily, including kubeconfig and network client paths in `pkg/kube` and `pkg/flagutil`, but these tag changes do not alter those source paths.
- **controller-runtime changes and exposure:** [v0.25.2 release notes](https://github.com/kubernetes-sigs/controller-runtime/releases/tag/v0.25.2) list a fix to `SubResourceCreateOptions.ApplyToSubResourceCreate`, new metrics-handler options, a ReadYourWrite caveat for multiple namespaces, and a ReadYourWrite wait-duration metric. Prow imports controller-runtime broadly, but no call to the fixed subresource helper or configuration enabling ReadYourWrite consistency was found. Production manager configurations set metrics `BindAddress` to `0` (for example, `cmd/tide/main.go:169`), so the new handler options are not enabled. No listed change is a security fix.
- **OSV for bumped modules:** queries for old and new versions returned no advisories for all six modules.
- **Repository-level govulncheck:** `govulncheck -format json ./...` exited successfully on both base and head. Both runs reported the same seven advisory IDs; the bump introduced or fixed none. Called traces were reported for `GO-2023-1901` in `github.com/tektoncd/pipeline v1.16.0` (32 traces) and five HTTP/2 advisories in `golang.org/x/net v0.59.0` (201 traces total). The x/net advisories are [GO-2026-6603](https://osv.dev/vulnerability/GO-2026-6603), [GO-2026-6610](https://osv.dev/vulnerability/GO-2026-6610), [GO-2026-6611](https://osv.dev/vulnerability/GO-2026-6611), [GO-2026-6612](https://osv.dev/vulnerability/GO-2026-6612), and [GO-2026-6617](https://osv.dev/vulnerability/GO-2026-6617); OSV lists `golang.org/x/net v0.60.0` as fixed. `GO-2026-5932` in `golang.org/x/crypto v0.57.0` was imported-only. These findings are unchanged by this PR.

## Open questions

- Is the roughly 9–10 day age of `controller-runtime v0.25.2` acceptable, or should this grouped bump wait until it has two weeks of soak?
- Are the reachable `x/net` and Tekton advisories tracked for a separate dependency update? This PR leaves them unchanged.

## Dependency followups

No actionable opportunities were identified. Examined `k8s.io/api`, `k8s.io/apimachinery`, `k8s.io/client-go`, `k8s.io/streaming`, and `k8s.io/code-generator` from `v0.37.0` to `v0.37.1`, plus `sigs.k8s.io/controller-runtime` from `v0.25.1` to `v0.25.2`. The Kubernetes module tag changes align dependency metadata without changing package source. Controller-runtime’s new metrics options and ReadYourWrite metric have no enabled Prow call site; its subresource create-options fix is not called by Prow. The changelog therefore exposes no concrete migration, replacement, or feature adoption for Prow-owned code.
