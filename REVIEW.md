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

## Dependency followups

Recorded 2026-10-19: 2 accepted, 0 skipped. PR #916 merged on 2026-09-07 as `93f38c01374ab03159d0d195cd57c05f708f4120`. Implement on the current default branch, checking whether subsequent changes already addressed each item. These are followup decisions, not a new review of the current default branch.

### Adopt typed informer handlers

- Category: api-upgrade
- Necessity: should
- Mode: deliver
- Status: accepted
- Dependency: `k8s.io/client-go v0.36.4 → v0.37.0`, with `k8s.io/code-generator v0.36.4 → v0.37.0` generating the typed informer wrappers.
- Where: `cmd/pipeline/controller.go:154`, `cmd/pipeline/controller.go:183`, `pkg/metrics/prowjobs/collector.go:72`; informer parameter declarations in `cmd/pipeline/controller.go:78` and `cmd/pipeline/main.go:80`; exporter caller in `cmd/exporter/main.go:110`.
- Source: [client-go v0.37.0 changelog](https://github.com/kubernetes/client-go/blob/v0.37.0/CHANGELOG.md#type-safe-informers): "event handlers are called with objects already cast to the correct type."
- Benefit: remove manual `any` assertions and explicitly handle deletion tombstones; the existing pipeline handlers reject tombstones rather than enqueueing the deleted resource.

```text
In kubernetes-sigs/prow, implement a followup enabled by PR #916 — "chore(deps): bump the kubernetes group across 2 directories with 4 updates" — which bumped k8s.io/client-go and k8s.io/code-generator from v0.36.4 to v0.37.0. The PR merged as 93f38c01374ab03159d0d195cd57c05f708f4120. Work on the current default branch; first check whether later changes already implement this migration.

Migrate the ProwJob and PipelineRun event registrations in cmd/pipeline/controller.go (reviewed lines 154 and 183) and the lifecycle-metrics update registration in pkg/metrics/prowjobs/collector.go (reviewed line 72) from Informer()/cache.ResourceEventHandlerFuncs with any callbacks to the generated TypedInformer() and typed handler APIs. The v0.37.0 changelog documents typed callbacks and identifies DeletedFinalStateUnknown handling as a motivation: https://github.com/kubernetes/client-go/blob/v0.37.0/CHANGELOG.md#type-safe-informers.

Update the necessary informer parameter types in controllerOptions, pipelineConfig, and the lifecycle collector, together with callers such as cmd/exporter/main.go. Use the generated ProwJobHandlerFuncs/PipelineRunHandlerFuncs and their DeletedProwJob/DeletedPipelineRun callback types where appropriate. Inspect the actual v0.37 API before implementing: deletion values can contain a key without an object. For object-bearing deletes, preserve the existing ProwJob cluster and Spec.Namespace routing. For key-only tombstones, use available namespace/name and cluster information when sufficient; when ProwJob routing cannot be determined safely, handle and report that case explicitly rather than guessing. Preserve normal enqueue keys, update semantics, resync handling, and histogram labels/observations.

Acceptance criteria: these registrations use typed callbacks and no longer need manual object assertions; focused tests cover normal add/update/delete events, object-bearing tombstones, key-only tombstones, and unchanged metrics. Run go test ./cmd/pipeline ./pkg/metrics/prowjobs ./cmd/exporter and make verify-codegen. Report any routing limitation of key-only ProwJob tombstones explicitly.

Scope guard: do not hand-edit generated informer files, upgrade dependencies, migrate unrelated controllers, redesign the workqueue, or change job reconciliation or metric semantics beyond the named deletion-event handling.
```

### Replace deprecated CRD required markers

- Category: deprecation
- Necessity: should; both forms remain supported and no removal deadline was announced.
- Mode: deliver
- Status: accepted
- Dependency: `sigs.k8s.io/controller-tools v0.21.0 → v0.22.0`.
- Where: `pkg/apis/prowjobs/v1/types.go:146`, `pkg/apis/prowjobs/v1/types.go:158`, `pkg/apis/prowjobs/v1/types.go:1119` — `ProwJobSpec.Type`, `ProwJobSpec.Job`, and `ProwJobStatus.State`.
- Source: [controller-tools v0.22.0 release](https://github.com/kubernetes-sigs/controller-tools/releases/tag/v0.22.0) deprecates the long required/optional marker forms in favor of `+required`/`+optional`, retaining compatibility.
- Benefit: keep owned API declarations on the supported preferred spelling without changing validation.

```text
In kubernetes-sigs/prow, implement a followup enabled by PR #916 — "chore(deps): bump the kubernetes group across 2 directories with 4 updates" — which bumped sigs.k8s.io/controller-tools from v0.21.0 to v0.22.0. The PR merged as 93f38c01374ab03159d0d195cd57c05f708f4120. Work on the current default branch; first check whether subsequent work already replaced these markers.

Replace the three +kubebuilder:validation:Required markers on ProwJobSpec.Type, ProwJobSpec.Job, and ProwJobStatus.State in pkg/apis/prowjobs/v1/types.go (reviewed lines 146, 158, and 1119) with +required. The v0.22.0 release deprecates the long spelling while retaining compatibility: https://github.com/kubernetes-sigs/controller-tools/releases/tag/v0.22.0. Preserve every JSON tag, enum, field type, and existing validation constraint.

Run make update-codegen and inspect the output. Acceptance criteria: all three declarations use +required; the generated ProwJob CRD has identical required-field constraints, including required spec.type, spec.job, and status.state; generated output has no unintended semantic changes; make verify-codegen passes. Use the shortest direct comparison of the regenerated schema against the pre-change version; no new test suite is needed for this marker-only migration.

Scope guard: do not modify API optionality or requiredness, rename fields, change JSON tags or enum constraints, migrate unrelated markers, or bump dependencies. Report unrelated generator churn rather than folding it into this cleanup.
```
