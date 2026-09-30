---
issue: kubernetes-sigs/prow#609
title: "Pipeline controller deletes PipelineRuns when two cluster contexts share the same cluster"
state: closed
labels: kind/bug, lifecycle/rotten
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-26T23:45:22Z
verdict: accepted
---

## Findings

### [cause] Context-name routing assumes 1:1 context-to-cluster identity
- detail: The pipeline controller creates one informer per kubeconfig context. When two contexts (e.g. `default` and a named alias) resolve to the same physical cluster/API server, both informers watch the same `PipelineRun` resources. The reconcile check treats a context-name mismatch as "wrong cluster" even when the underlying cluster is identical, so it deletes the PipelineRun.
- evidence: `cmd/pipeline/controller.go:463` — `ClusterToCtx(pj.Spec.Cluster) != ctx` evaluates true across the duplicate contexts, setting `wantPipelineRun = false`.

### [reproducibility] Reporter's multi-cluster migration scenario
- detail: Reporter configured `cluster-a`/`cluster-b`/`cluster-c` plus `default`, then pointed a ProwJob's `cluster:` at a named context sharing the same kubeconfig as `default` (mid-migration). Both contexts got independent informers on the same cluster, triggering the deletion bug. Matches the fix author's own PR description almost verbatim.
- evidence: issue body (https://github.com/kubernetes-sigs/prow/issues/609) and PR #749 description.

### [related-code] Existing dedup precedent used as template for the fix
- where: `cmd/pipeline/main.go:127-136` (pre-fix line numbers, per original 2026-02-10 triage)
- excerpt: |
    InClusterContext -> DefaultClusterAlias aliasing handled the empty-string vs "default" case specifically, but not the general case of any two contexts sharing a server URL.

### [related-pr] Fix: reject duplicate cluster endpoints at startup
- ref: kubernetes-sigs/prow#749
- relevance: Implements the originally recommended fix (startup validation). Adds `validateUniqueClusters(configs map[string]rest.Config) error` in `cmd/pipeline/main.go`, which maps `Host -> []contextName` and fails fast with `logrus.Fatal` if any host has more than one context. Includes `TestValidateUniqueClusters` (unique/duplicate/empty/single-config cases). 21 lines in `main.go`, 43 in `main_test.go`. Merged 2026-06-16T17:45:39Z, closed #609 via `Fixes: #609`. Verified: PR #749's merge commit `0f4e9fead` is an ancestor of this triage's `main_sha`, so the fix is live on main.

## Checked

- Confirmed `validateUniqueClusters` is present and wired into `main()` in the current worktree's `cmd/pipeline/main.go` (post-fix).
- Confirmed via `git merge-base --is-ancestor` that PR #749's merge commit is an ancestor of `main_sha` above.
- Reviewed all issue comments and timeline events after the 2026-06-10 `lifecycle/rotten` label — only new activity is PR #749's cross-reference and merge/close; no explanatory close comment (closed automatically via the PR's `Fixes:` keyword, not the lifecycle bot — rotten-driven auto-close would have landed around 2026-07-10, after the actual close date of 2026-06-16).
- Compared PR #749's diff against the originally recommended "Approach 1" (startup validation and deduplication) — matches directly, same file, same technique.

## Next steps

- None — issue is closed and fixed on main (`cmd/pipeline/main.go`, via PR #749). No labels, comments, or code changes outstanding.

## Open questions

- None.
