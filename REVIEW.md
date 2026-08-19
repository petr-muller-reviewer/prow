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
