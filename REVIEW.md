---
pr: kubernetes-sigs/prow#731
title: "Bump Kubernetes dependencies to v0.34.x"
head_sha: 8c59d6e493bd17d164fb35b1fb1ec9e85b0f6293
base: main
reviewed_at: 2026-06-23T23:56:03Z
verdict: needs-discussion
---

## Findings

### [question] Race detector CI failure
- where: CI job `pull-prow-unit-test-race-detector-nonblocking`
- concern: Race detector test is failing. The sidecar test restructuring (`pkg/sidecar/run_test.go`) is the most likely cause since it changes goroutine synchronization order. Needs investigation whether this is the cause or a pre-existing flake before merge.

### [should-fix] Sidecar test race fix — verify correctness
- where: `pkg/sidecar/run_test.go:243-281`
- concern: The restructuring changes the `missing` marker test case to create markers before starting `wait()`, then starts `wait()` and cancels. The intent is correct (eliminate race between marker creation and `wait()` startup), but the race detector is failing. The new flow calls `startWait()` after markers are written but before `cancel()` — there may be a race between `wait()` reading marker files and the `cancel()` call on the next line. The `time.Sleep(missingMarkerTimeout)` between `startWait()` and `cancel()` should help, but the timeout may be too short under race detector instrumentation.
- excerpt: |
    if tc.missing {
        startWait()
        time.Sleep(missingMarkerTimeout)
        cancel()
    }

### [question] CRD regeneration method
- where: `config/prow/cluster/prowjob-crd/prowjob_customresourcedefinition.yaml`
- concern: Large CRD changes include new fields (`fileKeyRef`, `restartPolicyRules`, `hostnameOverride`, `podCertificate`) and description updates (e.g. podAntiAffinity "adding weight" changed to "subtracting weight"). Need confirmation these were produced by running the standard CRD generator against the new API types, not hand-edited.

### [nit] Inconsistent diff package migration not documented
- where: `pkg/git/v2/executor_test.go:27`, `pkg/plugins/retitle/retitle_test.go:24`, `pkg/simplifypath/simplify_test.go:23`
- concern: Three files switched import from `k8s.io/apimachinery/pkg/util/diff` to `k8s.io/utils/diff` while the rest stayed on the apimachinery package. The reason is that these files use `StringDiff`/`ObjectReflectDiff` which were removed from apimachinery but exist in utils. Correct but undocumented — a PR description note would help future reviewers.

## Checked
- All `diff.ObjectReflectDiff` / `diff.ObjectDiff` to `diff.Diff` migrations are correct — the new `Diff` function exists in `k8s.io/apimachinery/pkg/util/diff` v0.34
- Three files switching to `k8s.io/utils/diff` use only `StringDiff` and `ObjectReflectDiff`, both available there
- YAML fixture updates (`preferences: {}` removal, `creationTimestamp: null` removal) match expected v0.34 serialization behavior
- `go.mod` / `go.sum` changes are consistent — no stale entries, structured-merge-diff v4 replaced by v6
- Transitive dependency bumps (fsnotify, cbor, go-restful, gnostic-models, reflect2) are reasonable
- No production code changes — all Go changes are in test files
- No security concerns in the dependency updates

## Open questions
- Was the CRD file regenerated with standard tooling (e.g. `make update` or controller-gen), or were changes applied manually?
- Is the race detector failure from the sidecar test fix or a pre-existing flake? Has `/retest` been attempted?
- The podAntiAffinity description change from "adding weight to the sum" to "subtracting weight from the sum" is an upstream Kubernetes API change — is this intentional upstream behavior change or a documentation correction?
