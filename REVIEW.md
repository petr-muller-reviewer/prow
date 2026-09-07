---
pr: kubernetes-sigs/prow#916
title: "chore(deps): bump the kubernetes group across 2 directories with 4 updates"
head_sha: 716b1d7b53adbd283f6eed96402430caf102af72
base: main
reviewed_at: 2026-09-07T17:29:55Z
verdict: approve
---

## Findings

None. The earlier generated-code failure from the dependency bump is addressed by commit `5d35e38d2` (`make update-codegen`).

## Checked

- Dep-only PR: root Kubernetes runtime dependencies move to v0.37.0; `hack/tools` moves `k8s.io/code-generator` to v0.37.0 and `sigs.k8s.io/controller-tools` to v0.22.0.
- Kubernetes v0.37.0 is a tagged release from 2026-08-26 (12 days of soak); controller-tools v0.22.0 is tagged from 2026-08-31 (7 days).
- Runtime Kubernetes APIs are heavily used, including cluster clients, informers, watches, admission decoding, and API objects. No incompatible project call site was found in the changed upstream API surface.
- `make update-codegen` regenerated the Prow and Tekton clients/informers, ProwJob CRD, and plugin documentation.
- `make verify-codegen` passes after regeneration.
- `git diff --check` passes.

## Open questions

None.
