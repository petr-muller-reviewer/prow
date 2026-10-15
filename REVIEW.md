---
pr: kubernetes-sigs/prow#999
title: "Restart PCM when disabled clusters change"
head_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
base: main
reviewed_at: 2026-10-09T15:53:10Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-10T16:01:08Z
  gated_head_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
  reviewed_head_sha: e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6
---

# Review

## Gate

**Decision: merge**

The PR remains open at the reviewed head; there are no new commits since review. The local review contains no gating findings, and GitHub has no submitted reviews, substantive inline comments, or actionable holds. The independent pass found no API or configuration compatibility changes.

**Gating list:** None.

**Independent merge risk:** `cmd/prow-controller-manager/main.go:133-140,259-269` adds a restart when disabled-cluster membership changes. Existing Kubernetes deployments will restart PCM on this config change; the sample deployment runs one replica, so controller activity can pause briefly. Jobs may still fail during the transition, as the PR description documents. This is the intended behavior, requires no migration, and is not an unacceptable merge risk. No separate release-note file changed; the PR description documents the behavior. It is not opt-in or rollout-gated.

## Verdict

Approve. Code quality, maintainability, and deployment risk reviewers all found no actionable issues. The change uses PCM's existing graceful shutdown path to rebuild clients and watches that are initialized only at startup, and introduces no configuration or API compatibility changes.

## What this PR does

- Subscribes PCM to config updates before initializing cluster clients.
- Compares old and new disabled-cluster membership as sets.
- Requests graceful shutdown when membership changes so the process restarts with rebuilt clients and pod watches.
- Adds focused tests for enabling, disabling, replacing, and equivalent disabled-cluster sets, plus a canceled watcher.

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

Accepted: 3. Skipped: 0. PR #999 was still open at head `e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6` when these followups were identified. Carry them out after the PR merges, from the updated default branch.

### Restart Sinker when disabled clusters change

- Category: reliability.
- Necessity: should — restore cleanup alongside job execution.
- Where: `cmd/sinker/main.go:114-119,181-207`; restart policy introduced in `cmd/prow-controller-manager/main.go:131-140,241-272`.
- Why followup: Sinker also filters and constructs build-cluster clients only at startup. After a cluster is re-enabled, PCM can create pods while Sinker's cleanup client remains absent until restart. This pre-existing gap is outside PR #999's PCM scope.

```text
In kubernetes-sigs/prow, following PR #999 — "Restart PCM when disabled clusters change" — make Sinker restart gracefully when disabled_clusters membership changes. The PR was open at head e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6 when this task was recorded; confirm it has merged and work from the updated default branch.

Inspect cmd/sinker/main.go, especially the config initialization at lines 114-119 and the build-cluster/client setup at lines 181-207. Sinker captures disabled clusters and constructs cleanup clients once. Re-enabling a cluster can therefore restore PCM job execution without restoring Sinker's cleanup for that cluster.

Subscribe to configuration changes before initializing cluster clients and request graceful shutdown when disabled-cluster set membership changes. Follow the policy established in cmd/prow-controller-manager/main.go: enabling, disabling, or replacing a member triggers a restart; unrelated config updates, reordering, duplicates, and equivalent nil/empty lists do not. Preserve cancellation handling and --run-once behavior. Reuse the policy without introducing a general restart framework.

Acceptance criteria: focused tests cover membership changes, equivalent sets, unrelated updates, and shutdown; the subscription is registered before client initialization; Sinker can rebuild its cleanup clients on the next process start; existing --run-once behavior is preserved. Run the relevant tests and report results.

Scope: Sinker configuration-triggered restart behavior only. Do not change PCM's behavior, other components, cleanup policies, or implement dynamic client/cache replacement.
```

### Document PCM's automatic restart and transition behavior

- Category: docs.
- Necessity: should — make the operational behavior discoverable after merge.
- Where: `site/content/en/docs/components/core/prow-controller-manager.md:31`; `pkg/config/config.go:237-239`; generated `pkg/config/prow-config-documented.yaml:442-445`.
- Why followup: The PR description explains automatic recovery and the remaining job failure window, but maintained operator documentation and the configuration field comment do not. Documentation can follow separately without delaying the recovery fix.

```text
In kubernetes-sigs/prow, following PR #999 — "Restart PCM when disabled clusters change" — document PCM's automatic restart behavior and its recovery limits. The PR was open at head e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6 when this task was recorded; confirm it has merged and work from the updated default branch.

Update site/content/en/docs/components/core/prow-controller-manager.md with operator guidance for disabled_clusters changes. Explain that membership changes initiate graceful shutdown so the container supervisor restarts PCM and rebuilds clients, caches, and pod watches. State the restart requirement for the deployment; equivalent sets, reordering, duplicates, nil/empty equivalence, and unrelated config changes do not trigger this restart.

Explain that configuration propagation, job assignment, and shutdown are not coordinated: jobs can still receive terminal missing-client errors during the transition, and PCM's restart does not retry jobs already marked errored. Describe the behavior present on the default branch if a later followup has changed these limits. Make clear that PR #999 adds this automatic recovery to PCM; do not imply all Prow components reload their cluster clients automatically.

Clarify the DisabledClusters field comment in pkg/config/config.go if needed, and regenerate pkg/config/prow-config-documented.yaml through the repository's existing generator whenever that comment changes. Keep the existing kubeconfig-filtering semantics accurate.

Acceptance criteria: maintained docs describe restart triggers, deployment requirements, expected interruption, and job recovery limits; field comments and generated YAML agree; links and formatting follow nearby documentation conventions. Use existing documentation/generation checks where applicable.

Scope: operator documentation and the configuration field documentation only. Do not change runtime behavior or promise coordinated or failure-free transitions.
```

### Retry recoverable missing-client errors within a bounded window

- Category: reliability.
- Necessity: could — reduce lost jobs during recovery; this requires a separate error-policy change.
- Where: `pkg/plank/reconciler.go:374-389,810-815,854-872`; `pkg/flagutil/kubernetes_cluster_clients.go:217-232,553-557`.
- Why followup: PR #999 explicitly leaves a window where the old process can terminally fail a job before restarting. The reconciler classifies all missing clients as permanent. Distinguishing recoverable absence from invalid configuration changes reconciliation policy and exceeds the restart fix's scope.

```text
In kubernetes-sigs/prow, following PR #999 — "Restart PCM when disabled clusters change" — add bounded recovery for jobs whose configured, currently enabled build cluster temporarily has no usable client. The PR was open at head e54ddad9c32a9c4f6a01f5afeae7c094ba1a5ab6 when this task was recorded; confirm it has merged and work from the updated default branch.

Inspect pkg/plank/reconciler.go, especially terminal-error completion at lines 374-389 and missing-client paths at lines 810-815 and 854-872, plus related pod deletion/abortion paths. PR #999 restarts PCM to rebuild clients, but a job reaching the old process can still be marked errored permanently. Establish a narrow policy that requeues temporarily unavailable clients for configured and currently enabled clusters, with an explicit finite recovery window and a clear terminal outcome when that window expires.

Distinguish unknown aliases and deliberately disabled clusters from recoverable absence; preserve their terminal error behavior. Do not rely solely on KubernetesOptions.KnownClusters: pkg/flagutil/kubernetes_cluster_clients.go returns the startup snapshot after disabled clusters were filtered, so it cannot identify a newly re-enabled cluster by itself. Establish accurate configured-cluster identity without mutating shared client maps or caches while controllers run. Check for unnecessary build-ID allocation and duplicate pod creation when introducing retries.

Acceptance criteria: tests show a configured/enabled cluster with a temporarily absent client does not immediately complete its job as errored; recovery allows normal reconciliation; expiry produces a clear terminal result; unknown aliases and disabled clusters remain terminal; retries are bounded and do not create duplicate pods. Cover the affected job states and run the relevant tests. Document the selected recovery policy and its remaining limits without claiming a failure-free transition.

Scope: classification and bounded retry of recoverable missing-client errors. Do not dynamically replace clients or pod watches, rerun already errored jobs, or redesign cross-component job assignment/configuration propagation.
```
