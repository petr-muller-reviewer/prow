---
pr: kubernetes-sigs/prow#999
title: "Restart PCM when disabled clusters change"
head_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
base: main
reviewed_at: 2026-10-15T23:14:39Z
verdict: approve
refresh_log:
  - old_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
    new_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
    summary: "No code changes; PR merged after approval."
gate:
  decision: merge
  gated_at: 2026-10-10T16:01:08Z
  gated_head_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
  reviewed_head_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
---

# Review

## Gate

**Decision: merge**

At the time of this gate (2026-10-10), the PR was open at the reviewed head; no new commits or gating findings were present, and GitHub had no submitted reviews, substantive inline comments, or actionable holds. The independent pass found no API or configuration compatibility changes. The PR has since merged without code changes.

**Gating list:** None.

**Independent merge risk:** `cmd/prow-controller-manager/main.go:133-140,259-269` adds a restart when disabled-cluster membership changes. Existing Kubernetes deployments will restart PCM on this config change; the sample deployment runs one replica, so controller activity can pause briefly. Jobs may still fail during the transition, as the PR description documents. This is the intended behavior, requires no migration, and is not an unacceptable merge risk. No separate release-note file changed; the PR description documents the behavior. It is not opt-in or rollout-gated.

## Verdict

Approve. Code quality, maintainability, and deployment risk reviewers all found no actionable issues. The change uses PCM's existing graceful shutdown path to rebuild clients and watches that are initialized only at startup, and introduces no configuration or API compatibility changes.

## What this PR does

- Subscribes PCM to config updates before initializing cluster clients.
- Compares old and new disabled-cluster membership as sets.
- Requests graceful shutdown when membership changes so the process restarts with rebuilt clients and pod watches.
- Adds focused tests for enabling, disabling, replacing, and equivalent disabled-cluster sets, plus a canceled watcher.

Since previous review:
- No code changes: `e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6` → `e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6`; the diff is empty.
- `@petr-muller` submitted an APPROVED review at 2026-10-10T16:11:27Z; `kubernetes-prow[bot]` posted the approval notification at 2026-10-10T16:11:35Z.
- PR state changed from OPEN to MERGED.

## Findings

No findings.

## Checked

- Code quality: watcher control flow, shutdown handling, channel closure, set comparisons, and test coverage; reviewer verdict was APPROVE.
- Maintainability: scope, reuse of existing config and interrupt mechanisms, logging, and test structure; maintenance burden assessed LOW.
- Deployment risk: no configuration schema, API, or CLI changes; existing configs remain compatible; risk assessed LOW.
- Operators may see a brief PCM interruption on restart. Jobs may still fail during the transition because assignment, config propagation, and shutdown are not coordinated; this limitation is described in the PR.
- Tests were reviewed but not run during this review.

## Open questions

None.

## Followups

Accepted: 3. Skipped: 1. The three previously accepted followups remain outstanding on the updated default branch. The unrelated-config watcher test proposed in this run was skipped.

### Restart Sinker when disabled clusters change

- Category: reliability.
- Necessity: should — restore cleanup alongside job execution.
- Where: `cmd/sinker/main.go:114-119,181-207`; PCM restart policy in `cmd/prow-controller-manager/main.go:131-140,247-269`.
- Why followup: Sinker also constructs build-cluster cleanup clients once at startup. Re-enabling a cluster can let PCM create pods while Sinker still lacks a client for cleanup; this neighboring component was outside PR #999's scope.

```text
In kubernetes-sigs/prow, following PR #999 — "Restart PCM when disabled clusters change" — make Sinker restart gracefully when disabled-cluster membership changes. PR #999 merged as commit 0104d7885a156dcb408e87aa10f517ca3936b16a; start from the current default branch.

Inspect cmd/sinker/main.go, especially config setup around lines 114-119 and build-cluster/client setup around lines 182-210. Sinker captures disabled clusters and constructs cleanup clients once. Re-enabling a cluster can therefore restore PCM job execution without restoring Sinker's cleanup client until Sinker restarts.

Subscribe to config changes before initializing clients and request graceful shutdown when disabled-cluster set membership changes. Match PCM's policy: enable, disable, or replace a member triggers a restart; unrelated config updates, reordering, duplicates, and nil/empty equivalence do not. Preserve cancellation handling and --run-once behavior. Reuse the focused policy without creating a general restart framework.

Acceptance criteria: tests cover membership changes, equivalent sets, unrelated updates, and shutdown; the subscription precedes client initialization; Sinker rebuilds cleanup clients after restart; existing --run-once behavior remains intact. Run relevant tests and report results.

Scope: Sinker config-triggered restart only. Do not change PCM behavior, other components, cleanup policies, or implement dynamic client/cache replacement.
```

### Document PCM's automatic restart and transition behavior

- Category: docs.
- Necessity: should — make the operational behavior discoverable after merge.
- Where: `site/content/en/docs/components/core/prow-controller-manager.md`; `pkg/config/config.go:237-239`; generated `pkg/config/prow-config-documented.yaml:442-445`.
- Why followup: The PR description explains automatic recovery and the remaining job-failure window, but maintained operator documentation and the config field comment do not. Documentation can follow separately without delaying the recovery fix.

```text
In kubernetes-sigs/prow, following PR #999 — "Restart PCM when disabled clusters change" — document PCM's automatic restart behavior and recovery limits. PR #999 merged as commit 0104d7885a156dcb408e87aa10f517ca3936b16a; start from the current default branch.

Update site/content/en/docs/components/core/prow-controller-manager.md with operator guidance for disabled_clusters changes. Explain that membership changes initiate graceful shutdown so the container supervisor restarts PCM and rebuilds clients, caches, and pod watches. State the deployment restart requirement; equivalent sets, reordering, duplicates, nil/empty equivalence, and unrelated config changes do not trigger this restart.

Explain that configuration propagation, job assignment, and shutdown are not coordinated: jobs can still receive terminal missing-client errors during the transition, and restarting PCM does not retry jobs already marked errored. Describe the behavior on the default branch if later work changes these limits. Make clear that PR #999 adds this automatic recovery to PCM; do not imply all Prow components reload cluster clients automatically.

Clarify the DisabledClusters field comment in pkg/config/config.go if needed, and regenerate pkg/config/prow-config-documented.yaml through the existing generator if that comment changes. Keep kubeconfig-filtering semantics accurate.

Acceptance criteria: maintained docs describe restart triggers, deployment requirements, expected interruption, and job-recovery limits; field comments and generated YAML agree; links and formatting follow nearby documentation conventions. Use existing documentation/generation checks where applicable.

Scope: operator and configuration-field documentation only. Do not change runtime behavior or promise coordinated or failure-free transitions.
```

### Retry recoverable missing-client errors within a bounded window

- Category: reliability.
- Necessity: could — reduce lost jobs during recovery; this is a separate error-policy change.
- Where: `pkg/plank/reconciler.go:374-389,810-815,854-872`; `pkg/flagutil/kubernetes_cluster_clients.go:217-232,553-557`.
- Why followup: PR #999 explicitly leaves a window where the old process can terminally fail a job before restarting. The reconciler classifies missing clients as permanent; distinguishing recoverable absence from invalid configuration changes reconciliation policy and exceeds the restart fix's scope.

```text
In kubernetes-sigs/prow, following PR #999 — "Restart PCM when disabled clusters change" — add bounded recovery for jobs whose configured, currently enabled build cluster temporarily has no usable client. PR #999 merged as commit 0104d7885a156dcb408e87aa10f517ca3936b16a; start from the current default branch.

Inspect pkg/plank/reconciler.go, especially terminal-error completion around lines 374-389 and missing-client paths around lines 810-815 and 854-872, plus related pod deletion/abortion paths. PR #999 restarts PCM to rebuild clients, but a job reaching the old process can still be marked errored permanently. Add a narrow policy that requeues temporary client unavailability for configured and currently enabled clusters, with a finite recovery window and a clear terminal outcome after expiry.

Distinguish unknown aliases and deliberately disabled clusters from recoverable absence; preserve their terminal error behavior. Do not rely only on KubernetesOptions.KnownClusters: pkg/flagutil/kubernetes_cluster_clients.go returns the startup snapshot after disabled clusters were filtered, so it cannot identify a newly re-enabled cluster by itself. Establish accurate configured-cluster identity without mutating shared client maps or caches while controllers run. Check for unnecessary build-ID allocation and duplicate pod creation during retries.

Acceptance criteria: tests show that a configured/enabled cluster with a temporarily absent client is not immediately completed as errored; recovery permits normal reconciliation; expiry produces a clear terminal result; unknown aliases and disabled clusters remain terminal; retries are bounded and do not create duplicate pods. Cover affected job states and run relevant tests. Document the recovery policy and remaining limits without claiming a failure-free transition.

Scope: classification and bounded retry of recoverable missing-client errors. Do not dynamically replace clients or pod watches, rerun already errored jobs, or redesign cross-component job assignment/config propagation.
```
