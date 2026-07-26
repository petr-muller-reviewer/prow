# Issue #609: Pipeline controller deletes PipelineRuns when two cluster contexts share the same cluster

**Status**: Triage Complete | **Verdict**: Legitimate | **Effort**: Level 1 — Easy | **Kind**: Bug  
**Author**: derryos | **Triaged**: 2026-02-10 | **Refreshed**: 2026-06-10  
**Labels (current)**: `kind/bug`, `lifecycle/rotten`  
**Issue**: https://github.com/kubernetes-sigs/prow/issues/609

---

## Issue Summary

The reporter uses multiple build clusters (`cluster-a`, `cluster-b`, `cluster-c`, plus `default`). They configured a named cluster with the same kubeconfig as the default cluster, intending to transition away from `default` post-upgrade. The pipeline controller gets confused because the default cluster and the named cluster share the same kubeconfig/context, causing ProwJobs to be erroneously deleted.

The specific code triggering the issue is in `cmd/pipeline/controller.go` around line 463.

---

## Root Cause Analysis

The pipeline controller assumes each kubeconfig context name maps to a physically distinct cluster. When two context names point to the same underlying cluster, both contexts get independent informers that watch the same resources. During reconciliation, the context-name-based routing incorrectly identifies PipelineRuns as being in the "wrong cluster" because the context names differ, even though the underlying cluster is the same.

### Failure Mechanism

1. Both informers watch the same cluster and see the same PipelineRun events
2. ProwJob specifies `Cluster: "my-cluster"` — reconcile called with `ctx = "my-cluster"`
3. The `"default"` informer also enqueues the same ProwJob (same cluster), calling reconcile with `ctx = "default"`
4. Check at line 463: `ClusterToCtx(pj.Spec.Cluster) != ctx` evaluates to `"my-cluster" != "default"` → **true**
5. `wantPipelineRun = false` → controller **deletes the PipelineRun**

### Contributing Factors

- The `pipelines` map uses context name as key with no deduplication by cluster identity
- The `InClusterContext → DefaultClusterAlias` dedup in `main.go:127-136` handles only one alias case, not the general case
- The `getPipelineConfig` fallback to "default" (`controller.go:285-296`) masks configuration issues rather than failing fast
- No validation at startup that all configured contexts point to distinct clusters

---

## Key Code Paths

| File | Description |
|------|-------------|
| `cmd/pipeline/main.go:120-164` | Cluster config loading, iterates contexts, creates pipelineConfig per context |
| `cmd/pipeline/main.go:127-136` | Existing InClusterContext → DefaultClusterAlias dedup (template for the fix) |
| `cmd/pipeline/controller.go:181-195` | Event handler registration, one informer per context |
| `cmd/pipeline/controller.go:255-270` | Work queue key format: `context/namespace/name` |
| `cmd/pipeline/controller.go:285-296` | Pipeline config fallback to "default" |
| `cmd/pipeline/controller.go:463` | The check that triggers deletion: `ClusterToCtx(pj.Spec.Cluster) != ctx` |
| `pkg/kube/config.go:34-71` | Cluster config loading |
| `pkg/pjutil/pjutil.go:408-414` | ClusterToCtx: maps empty string to "default" |

---

## Proposed Solutions

### Approach 1: Startup Validation and Deduplication ⭐ Recommended — Low Complexity

At startup, detect when multiple kubeconfig contexts resolve to the same cluster API server. Reject the configuration with a clear error.

**Pros**: Prevents the problem entirely; fast-fail gives operators clear feedback; minimal changes to reconciliation logic; no runtime overhead  
**Cons**: May break existing (already-broken) configs; requires comparing REST config server URLs; deduplication must decide which context name to keep

Affected: `cmd/pipeline/main.go`, possibly `pkg/kube/config.go`

### Approach 2: Reconciliation-Level Tolerance — Medium Complexity

Modify reconciliation to skip processing when a PipelineRun is found via one context but belongs to a ProwJob targeting a different context that resolves to the same cluster.

**Pros**: Allows alias configurations; more flexible for migration; fixes the symptom directly  
**Cons**: Runtime overhead in hot path; more complex reconciliation logic; still creates redundant informers

### Approach 3: Warn at Startup, Skip at Reconciliation — Medium Complexity

Detect overlapping contexts at startup, log a warning, build a context→cluster mapping, and use it during reconciliation to skip non-primary contexts.

**Pros**: Best of both — warns AND prevents; no configuration breakage; mapping built once, cheap at runtime  
**Cons**: More code than either alone; must define "primary" selection logic

---

## Effort Assessment

| Factor | Assessment |
|--------|------------|
| Scope | Small — 1-2 files, ~30-50 lines |
| Complexity | Simple — map-based dedup logic |
| Expertise | Minimal — basic Go + kubeconfig |
| Clarity | Well-defined — no open design questions |
| Testing | Simple — follow existing patterns |
| Compatibility | Fully compatible — broken configs stay broken |
| Alignment | Perfect fit — extends existing pattern |
| Dependencies | None — pure Go comparisons |

### Recommended Labels

- `kind/bug` — destructive behavior (PipelineRun deletion) from valid-seeming configuration
- `help-wanted` — code change is small but reasoning about multi-cluster configurations is non-trivial

### Contributor Guidance

- Good starting point for new Prow contributors with basic Go and Kubernetes knowledge
- Review `cmd/pipeline/main.go:120-164` (template to follow) and `controller.go:463` (the bug)
- Implementation: after loading cluster configs, build a reverse map of `Host → []contextName`; if any Host has multiple contexts, error and exit
- Consider URL normalization (trailing slashes, port numbers) when comparing Host fields

---

## Test Coverage Gaps

Existing tests cover 23 reconcile cases and cross-cluster deletion/ignoring scenarios, but miss the core issue:

- No test for two contexts pointing to the same underlying cluster (same server URL, different context names)
- No test for event handler deduplication when contexts resolve to the same cluster
- No test for ProwJob routing correctness when a named alias points to the same backend as "default"

---

## Posted GitHub Comment

Posted: 2026-02-10 — https://github.com/kubernetes-sigs/prow/issues/609#issuecomment-3874545969

> /retitle Pipeline controller deletes PipelineRuns when two cluster contexts share the same cluster
>
> The root cause is in how the pipeline controller sets up per-cluster informers. When multiple kubeconfig contexts point to the same underlying cluster, each context gets its own independent informer watching the same resources. When a PipelineRun event arrives via the "wrong" context's informer, the reconciliation check at `cmd/pipeline/controller.go:463` (`ClusterToCtx(pj.Spec.Cluster) != ctx`) evaluates to true, setting `wantPipelineRun = false` and causing the controller to delete the PipelineRun. This only affects the pipeline (Tekton) controller; other Prow controllers operate in single-cluster mode and are not impacted.
>
> The fix would be to add startup validation in `cmd/pipeline/main.go` that detects when multiple kubeconfig contexts resolve to the same cluster API server endpoint. There's already precedent for this kind of deduplication in the same file: the `InClusterContext` → `DefaultClusterAlias` mapping at lines 127-136 handles the specific case of empty-string vs "default" aliasing. The general case (any two contexts sharing a server URL) can follow the same pattern. As a workaround until this is fixed, ensure each kubeconfig context name maps to a distinct cluster — remove the duplicate context or avoid configuring `cluster:` in ProwJobs to use a named alias that shares a cluster with `default`.
>
> /kind bug  
> /help-wanted

---

## Triage Timeline

- **2026-02-10** Initial Validation — assessed as LEGITIMATE bug
- **2026-02-10** Code Research — root cause identified: dual informers on same cluster → wrong-cluster check → PipelineRun deletion
- **2026-02-10** Effort Assessment — Level 1 (Easy), ~30-50 lines of validation code
- **2026-02-10** Issue Augmentation — comment drafted with root cause, fix approach, workaround, and labels
- **2026-02-10** Maintainer Briefing — briefed and approved
- **2026-02-10** Comment Posted — [issuecomment-3874545969](https://github.com/kubernetes-sigs/prow/issues/609#issuecomment-3874545969)

---

## Activity Since Triage

- **2026-02-10**: `kind/bug` applied by k8s-ci-robot (from triage comment). `/help-wanted` was not applied — not a recognized prow label in this repo; consider applying `help-wanted` manually if the label exists, or using the correct label name.
- **2026-05-11**: `lifecycle/stale` applied by k8s-triage-robot (90 days of inactivity).
- **2026-06-10**: `lifecycle/rotten` applied by k8s-triage-robot (30 days after stale). Issue will be **auto-closed around 2026-07-10** if no activity.

## Next Steps

- **Action required**: The issue is now `lifecycle/rotten`. Either `/remove-lifecycle rotten` to keep it open (if there's intent to fix), or let it close. Given it's a Level 1 good-first-issue candidate, it's worth keeping open.
- Consider creating an `area/pipeline` label for future pipeline controller issues
