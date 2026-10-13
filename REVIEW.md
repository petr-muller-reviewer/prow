---
pr: kubernetes-sigs/prow#853
title: "chore(deps): bump the kubernetes group across 1 directory with 2 updates"
head_sha: a6561e582ddc171130fa213eb90eab074615caeb
base: main
reviewed_at: 2026-08-19T11:42:30Z
verdict: approve
state: merged
refresh_log:
  - old_sha: a6561e582ddc171130fa213eb90eab074615caeb
    new_sha: a6561e582ddc171130fa213eb90eab074615caeb
    summary: No code changes. PR approved by petr-muller (2026-08-17T17:15:23Z), /lgtm'd (2026-08-17T17:27:36Z), auto-approved by OWNERS bot (2026-08-17T17:32:43Z), and merged.
---

## What this PR does

- Dependabot bump of `k8s.io/code-generator` 0.32.8 → 0.36.3 and `sigs.k8s.io/controller-tools` 0.17.3 → 0.21.0 in `hack/tools/go.mod` (build-tool-only module, not part of the shipped `sigs.k8s.io/prow` module).
- The bump alone broke `make update-codegen`: code-generator v0.36 removed the dead `--bounding-dirs` flag from `deepcopy-gen`, which `hack/make-rules/update/codegen.sh` relied on.
- Second commit (human-authored, Petr Muller) fixes `gen-deepcopy` in `codegen.sh` to pass target packages as positional args instead, then regenerates all affected output: `zz_generated.deepcopy.go`, clientset/lister/informer code under `pkg/client/...` and `pkg/pipeline/...`, and the CRD YAML annotation.
- All regenerated code is upstream-generated boilerplate (new context-aware `Start`/`WaitForCacheSync` variants, `InformerName`-based metrics identity, `errgroup`-style `wg.Go`) reflecting the new code-generator/controller-tools versions — no hand-written logic beyond the one-line generator invocation fix.

## Findings

### [nit] positional deepcopy-gen args don't recurse into future subpackages of pkg/apis
- where: `hack/make-rules/update/codegen.sh:95-104`
- concern: Old invocation used `--bounding-dirs sigs.k8s.io/prow/pkg/apis,sigs.k8s.io/prow/pkg/config`, which recursed into any subpackage under `pkg/apis`. New invocation lists `sigs.k8s.io/prow/pkg/apis/prowjobs/v1` explicitly. Today `pkg/apis` has exactly one subpackage (`prowjobs/v1`) so coverage is unchanged, but if a second API package is added under `pkg/apis/` in the future, it won't get DeepCopy generation automatically — a new positional entry would need to be added by hand.
- excerpt: |
    "$deepcopygen" \
      --go-header-file hack/boilerplate/boilerplate.generated.go.txt \
      --output-file zz_generated.deepcopy.go \
      sigs.k8s.io/prow/pkg/apis/prowjobs/v1 \
      sigs.k8s.io/prow/pkg/config

## Checked
- Both bumped deps (`k8s.io/code-generator`, `sigs.k8s.io/controller-tools`) are direct deps of `hack/tools/go.mod` only, a separate tool-only Go module; `grep -rn` outside `hack/tools` found no import of either in the shipped module. No runtime exposure.
- Release freshness: code-generator v0.36.3 released 2026-07-23 (~25 days old), controller-tools v0.21.0 released 2026-05-06 (~103 days old). Both fine, not compromised-release-risk-fresh.
- Confirmed upstream removal of `--bounding-dirs` from `deepcopy-gen` via code-generator commit history ("deepcopy-gen: remove dead --bounding-dirs flag") — matches the PR's stated rationale and the `Fixes: unknown flag: --bounding-dirs` trailer.
- `f.informerName.Release()` added in regenerated `pkg/client/informers/externalversions/factory.go` `Shutdown()`: verified nil-receiver-safe in client-go v0.36.3 (`tools/cache/identity.go`, `Release()` starts with `if n == nil { return }`), so factories that never call the new `WithInformerName` option (all current Prow call sites) are unaffected.
- CRD YAML diff (`config/prow/cluster/prowjob-crd/prowjob_customresourcedefinition.yaml`) is only the `controller-gen.kubebuilder.io/version` annotation bump — cosmetic.
- No CVEs or security advisories found in either dependency's compare range for the bumped versions.

## Open questions
- None.

## Since previous review
- No code changes (head SHA unchanged at `a6561e582ddc171130fa213eb90eab074615caeb`).
- petr-muller submitted an APPROVED review (2026-08-17T17:15:23Z) and commented `/lgtm` (2026-08-17T17:27:36Z).
- `kubernetes-prow[bot]` posted the OWNERS approval notifier marking the PR APPROVED, with approvers cblecker (self) and petr-muller (2026-08-17T17:32:43Z).
- PR state is now `MERGED`.

## Dependency followups

Examined at 2026-10-13T14:27:24Z. PR #853 merged at 2026-08-17T18:21:49Z as `406c912aaa81195a89912d15631f3836bb9936c2`. Accepted handoffs target the merged default branch. Two opportunities accepted; zero skipped.

### Scope examined

The upstream/main merge-base is already the PR head because the PR merged. The dependency inventory therefore uses the first PR commit’s parent, `7e14d333447e988af71131de0bd52db54cd14906`, through PR head `a6561e582ddc171130fa213eb90eab074615caeb`; the local review-artifact commit is excluded. All manifest/lock changes are in the separate `hack/tools` Go module. Its Go directive changes from 1.25.5 to 1.26.0.

| Module | Previous | New | Classification |
| --- | --- | --- | --- |
| `github.com/fsnotify/fsnotify` | `v1.9.0` | `v1.10.0` | indirect |
| `github.com/fxamacker/cbor/v2` | `v2.9.0` | `v2.9.1` | indirect |
| `github.com/gogo/protobuf` | `v1.3.2` | removed | indirect |
| `github.com/google/gnostic-models` | added | `v0.7.1` | indirect |
| `github.com/grpc-ecosystem/grpc-gateway/v2` | `v2.27.3` | `v2.27.7` | indirect |
| `google.golang.org/protobuf` | `v1.36.11` | `v1.36.12-0.20260120151049-f2248ac996af` | direct |
| `k8s.io/api` | `v0.35.0` | `v0.36.0` | indirect |
| `k8s.io/apiextensions-apiserver` | `v0.32.8` | `v0.36.0` | indirect |
| `k8s.io/apimachinery` | `v0.35.0` | `v0.36.3` | indirect |
| `k8s.io/code-generator` | `v0.32.8` | `v0.36.3` | direct |
| `k8s.io/klog/v2` | `v2.130.1` | `v2.140.0` | indirect |
| `k8s.io/kube-openapi` | `v0.0.0-20251125145642-4e65d59e963e` | `v0.0.0-20260427204847-8949caaa1199` | indirect |
| `k8s.io/utils` | `v0.0.0-20251002143259-bc988d571ff4` | `v0.0.0-20260319190234-28399d86e0b5` | indirect |
| `sigs.k8s.io/controller-tools` | `v0.17.3` | `v0.21.0` | direct |
| `sigs.k8s.io/structured-merge-diff/v6` | `v6.3.1` | `v6.4.0` | indirect |

Examined the complete [code-generator v0.32.8 → v0.36.3 commit range](https://github.com/kubernetes/code-generator/compare/v0.32.8...v0.36.3) and [controller-tools v0.17.3 → v0.21.0 commit range](https://github.com/kubernetes-sigs/controller-tools/compare/v0.17.3...v0.21.0), including the intermediate [v0.18.0](https://github.com/kubernetes-sigs/controller-tools/releases/tag/v0.18.0), [v0.19.0](https://github.com/kubernetes-sigs/controller-tools/releases/tag/v0.19.0), [v0.20.0](https://github.com/kubernetes-sigs/controller-tools/releases/tag/v0.20.0), [v0.20.1](https://github.com/kubernetes-sigs/controller-tools/releases/tag/v0.20.1), and [v0.21.0](https://github.com/kubernetes-sigs/controller-tools/releases/tag/v0.21.0) notes. Cross-referenced generator invocations, API markers, and handwritten consumers of generated factories.

Also examined imported transitive ranges: [fsnotify v1.9.0 → v1.10.0](https://github.com/fsnotify/fsnotify/compare/v1.9.0...v1.10.0), [protobuf v1.36.11 → f2248ac996af](https://github.com/protocolbuffers/protobuf-go/compare/v1.36.11...f2248ac996af), [k8s.io/api v0.35.0 → v0.36.0](https://github.com/kubernetes/api/compare/v0.35.0...v0.36.0), [apimachinery v0.35.0 → v0.36.3](https://github.com/kubernetes/apimachinery/compare/v0.35.0...v0.36.3), and [utils bc988d571ff4 → 28399d86e0b5](https://github.com/kubernetes/utils/compare/bc988d571ff4...28399d86e0b5). These versions changed in the tools module only. The runtime module already used the same or newer Kubernetes/protobuf versions before this PR, and still selects fsnotify v1.9.0; those ranges supply no runtime migration newly enabled by this bump.

Lock-only changes inventoried: gomega v1.39.1 → v1.40.0, otel/exporters/otlp/otlptrace v1.39.0 → v1.40.0, and x/exp 8475f28825e9 → 944ab1f22d93. Newly checksummed modules: cel.dev/expr v0.25.1, antlr/v4 v4.13.0, semver/v4 v4.0.0, cel-go v0.26.0, go-strcase v1.3.0, otlptracegrpc v1.40.0, k8s.io/apiserver/client-go/component-base v0.36.0, and konnectivity-client v0.34.0. Removed checksums also include gotool v1.0.0 and obsolete errcheck, goldmark, and x/tools versions. New checksums do not establish a runtime dependency upgrade: semver and client-go are imported by Prow but their main-module requirements did not change; goldmark only lost an obsolete checksum. Pure tool-transitive changes with no owned import or exposed generator feature were deprioritized.

No additional handoff: the removed deepcopy-gen flag is already addressed by the PR; the existing review nit remains in Findings. NewFilteredSharedInformerFactory has no handwritten callers to migrate. Named-informer metrics have no configured informer metrics provider in the examined consumers. Applyconfiguration, webhook/RBAC, one-of, immutability, and declarative-validation features have no concrete migration justified by these usages. ProwJobState enum migration was excluded because it would tighten validation at additional list/map fields.

### 1. Pass contexts to generated informer startup

- Category: `api-upgrade`.
- Necessity: `could`.
- Unlocked by: `k8s.io/code-generator v0.32.8 → v0.36.3`.
- Decision: accepted.
- Where: `cmd/exporter/main.go:102`; `cmd/pipeline/main.go:85,93,100,169,174`.
- Changelog: "Passing a context to StartWithContext enables context-aware reflector logging." [Upstream commit](https://github.com/kubernetes/code-generator/commit/d8c5f7828bcf31c7ea6577c6814b947b97aef734).

```text
In kubernetes-sigs/prow, work from the current default branch containing merge commit 406c912aaa81195a89912d15631f3836bb9936c2. PR #853, "chore(deps): bump the kubernetes group across 1 directory with 2 updates", upgraded k8s.io/code-generator v0.32.8 → v0.36.3 and regenerated the ProwJob and Tekton informer factories.

Use the new StartWithContext API at the three existing startup sites: cmd/exporter/main.go:102 and cmd/pipeline/main.go:100,169. Call it synchronously because it already launches the informer goroutines and returns. Change newPipelineConfig (cmd/pipeline/main.go:85) to accept context.Context, pass the shutdown context from its caller at line 174, and use that context for the initial PipelineRuns List request at line 93. Upstream explains that passing a context to StartWithContext enables context-aware reflector logging: https://github.com/kubernetes/code-generator/commit/d8c5f7828bcf31c7ea6577c6814b947b97aef734.

Acceptance criteria: all three generated-factory startup sites use StartWithContext; the pipeline setup List and informer share the supplied context; cancellation stops the setup request and informers; preserve informer registration before startup, namespace filtering, resync intervals, cluster selection and existing shutdown behavior. Run focused existing checks for cmd/exporter and cmd/pipeline, adding a targeted cancellation test only where needed.

Scope: change the callers and helper signature. Keep controller.Run, its cache-sync mechanism, metrics, generated files and dependency versions outside this task.
```

### 2. Derive the ProwJob type enum from Go constants

- Category: `simplification`.
- Necessity: `could`.
- Unlocked by: `sigs.k8s.io/controller-tools v0.17.3 → v0.21.0`.
- Decision: accepted.
- Where: `pkg/apis/prowjobs/v1/types.go:38,43-49,145-147`; generated ProwJob CRD.
- Changelog: "Add support for k8s:enum markers." [v0.21.0 release](https://github.com/kubernetes-sigs/controller-tools/releases/tag/v0.21.0).

```text
In kubernetes-sigs/prow, work from the current default branch containing merge commit 406c912aaa81195a89912d15631f3836bb9936c2. PR #853, "chore(deps): bump the kubernetes group across 1 directory with 2 updates", upgraded sigs.k8s.io/controller-tools v0.17.3 → v0.21.0. Its new type-level +k8s:enum marker derives enum values from typed string constants (https://github.com/kubernetes-sigs/controller-tools/pull/1352; https://github.com/kubernetes-sigs/controller-tools/releases/tag/v0.21.0).

In pkg/apis/prowjobs/v1/types.go, add +k8s:enum to ProwJobType (line 38), whose four typed constants are at lines 43–49. Remove the duplicated +kubebuilder:validation:Enum marker from ProwJobSpec.Type (line 145), preserving the Required marker. Regenerate config/prow/cluster/prowjob-crd/prowjob_customresourcedefinition.yaml through the existing generation workflow.

Acceptance criteria: the generated spec.type enum still contains exactly presubmit, postsubmit, periodic and batch; its requiredness and all other schema validation remain unchanged. Enum ordering may change because the generator sorts derived values. Inspect the CRD diff and run the relevant code-generation verification. Confirm that a future typed ProwJobType constant will supply its enum value without a second string list.

Scope: limit the migration to ProwJobType and the derived CRD. Leave ProwJobState, ProwJobAgent, runtime behavior, TypeScript definitions and dependency versions outside this task. ProwJobState also appears in JobStatesToReport and PrevReportStates; marking that type would add validation to currently unrestricted list/map values and requires a separate compatibility assessment.
```
