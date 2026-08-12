---
pr: kubernetes-sigs/prow#845
title: "chore(deps): bump the kubernetes group across 1 directory with 2 updates"
head_sha: 0e3a2a2fbfeab2ee2540bea3675da904f1267683
base: main
reviewed_at: 2026-08-12T11:24:49Z
verdict: approve
---

## What this PR does
- Dependabot group bump in `hack/tools/go.mod`/`go.sum` only (build-tooling module, separate from the main Prow module).
- Direct deps: `k8s.io/code-generator` v0.32.8 → v0.36.3 (released 2026-07-23, ~20d old), `sigs.k8s.io/controller-tools` v0.17.3 → v0.21.0 (released 2026-05-06, ~3mo old).
- `go` directive 1.25.5 → 1.26.0; long tail of transitive bumps pulled in by the above (k8s.io/api, apimachinery, apiextensions-apiserver → 0.36.x; kube-openapi, klog, utils, structured-merge-diff/v6, fsnotify, cbor, grpc-gateway; `github.com/gogo/protobuf` dropped, `github.com/google/gnostic-models` added; `google.golang.org/protobuf` moved to pseudo-version `v1.36.12-0.20260120151049-f2248ac996af`).
- Both direct deps are used only as build-time CLI binaries (`client-gen`, `deepcopy-gen`, `informer-gen`, `lister-gen`, `controller-gen`) invoked from `hack/make-rules/update/codegen.sh:40-48`; nothing here is imported into any shipped Prow binary.
- Classification: dep-only (no project source, no vendored/generated artifacts touched).

## Findings

None. Dep-only bump, no project code changed.

## Checked
- Diff scope: only `hack/tools/go.mod` and `hack/tools/go.sum` changed — confirmed dep-only.
- Freshness of both direct deps via proxy.golang.org: code-generator v0.36.3 (2026-07-23), controller-tools v0.21.0 (2026-05-06) — both outside the risky <2wk window.
- Import surface: grepped for `controller-tools`/`code-generator` usage outside go.mod/go.sum — only referenced as CLI builds in `hack/make-rules/update/codegen.sh`, not imported into Prow source.
- Changelog for both deps (code-generator v0.32.8..v0.36.3, controller-tools v0.17.3..v0.21.0): additive generator features (declarative-validation markers, k8s:enum/k8s:immutable, applyconfiguration extract functions, kube-openapi/k8s.io v1.36 bump) — no CVEs, no runtime exposure since these tools never execute against untrusted input in Prow.

## Open questions
- Given the 4-minor-version jump on both generators (code-generator, controller-tools) and the markers/CRD-generation changes in that range (e.g. IntOrString enum handling, cluster-scoped applyconfiguration fix), has `make update-codegen`/`verify-codegen` been re-run to confirm no unexpected diff in checked-in generated code?
