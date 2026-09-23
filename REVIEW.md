---
pr: kubernetes-sigs/prow#804
title: "plank: revive pods terminated by kubelet"
head_sha: e039405950fc1f0303e1388925b839aab7016f60
base: main
reviewed_at: 2026-09-23T19:16:06Z
verdict: needs-discussion
---

# Review

## Verdict

Needs discussion.

The implementation is small, scoped to a precise Kubernetes condition, and reuses Plank's bounded revival path; code-quality, maintainability, and deployment-risk passes found no implementation blocker. Merge should wait for maintainer agreement on the intended policy: in particular, whether a `TerminationByKubelet` pod should be retried by default and whether the proposed log-only/opt-in rollout is required.

## What this PR does

- Adds the `termination-by-kubelet` unexpected-stop cause when a terminated pod has an active `DisruptionTarget` condition with reason `TerminationByKubelet`.
- Sends that pod through Plank's existing delete-and-recreate path rather than completing its ProwJob as a test failure.
- Reuses the existing `MaxRevivals` cap; it adds no configuration, CRD, RBAC, or API surface.
- Adds predicate-level negative cases and a controller test for the revival path.
- Since the prior review, no code has changed. Maintainers discussed an opt-in or log-only rollout; `mimowo` proposed collecting decisions for a month before enabling retries, `petr-muller` endorsed that approach, and the author asked to merge without an accompanying implementation change.

## Findings

### [question] default revival policy remains unresolved
- where: PR discussion, comments from 2026-07-27 through 2026-09-17
- concern: `Prucek` canceled LGTM over retrying `TerminationByKubelet` pods, particularly node-pressure evictions. `mimowo` argues Kubernetes only sets `DisruptionTarget` for retriable disruptions and that the existing revival cap bounds impact. The later proposed log-only or opt-in rollout is not implemented, so maintainers still need an explicit policy decision before merge.
- excerpt: |
    mimowo: "introduce the code but only log the decisions - skip retrying"
    ... "observe for a month"

### [question] scope of DisruptionTarget handling is undecided
- where: `pkg/plank/reconciler.go:714-728`
- concern: `mimowo` proposed generalizing the predicate to any `DisruptionTarget` reason, while `droslean` asked for validation of non-Kubernetes/OpenShift behavior first. This PR intentionally remains `TerminationByKubelet`-specific, but the intended scope should be agreed before it becomes the default behavior.
- excerpt: |
    if podWasDisrupted(pod) {
        return PodUnexpectedStopCauseDisruption
    }

### [question] decide whether strict jobs need an opt-out
- where: `pkg/plank/reconciler.go:487` and `pkg/plank/reconciler.go:682-711`
- concern: `ErrorOnEviction` already lets a job fail instead of being retried after eviction. The new cause has no comparable per-job opt-out, so matching jobs always consume their revival budget. This is not an implementation defect, but it is the concrete policy choice an opt-in rollout would address.
- excerpt: |
    if podUnexpectedStopCause == PodUnexpectedStopCauseEvicted && pj.Spec.ErrorOnEviction {
        return nil
    }

### [nit] cover the revival-limit boundary for the new cause
- where: `pkg/plank/controller_test.go:TestSyncPendingJob`
- concern: The controller test confirms the new cause deletes and revives a pod, while the existing eviction test covers `PodRevivalCount >= MaxRevivals`. An analogous new-cause case would make the key safety bound explicit and guard against future control-flow changes.
- excerpt: |
    case pj.Status.PodRevivalCount >= *r.config().Plank.MaxRevivals:
        // MaxRevivals is reached, complete the PJ and mark it as errored.

## Checked

- `podWasTerminatedByKubelet` uses the upstream `corev1.DisruptionTarget` and `corev1.PodReasonTerminationByKubelet` constants rather than literal strings.
- The predicate requires `status.reason == Terminated`, an active `DisruptionTarget`, and the exact condition reason, avoiding ordinary failed-test pods.
- Predicate tests cover the matching case and inactive, wrong-reason, missing-condition, and non-terminated near misses.
- The controller test verifies the matching pod is deleted and the ProwJob stays pending for revival.
- Existing `MaxRevivals` enforcement bounds replacement attempts; no configuration compatibility, RBAC, or new external dependency risk was introduced.

## Open questions

- Should this ship as the current default, an opt-in behavior, or a log-only observation phase first?
- Should the implementation stay specific to `TerminationByKubelet`, or be broadened to all `DisruptionTarget` reasons after validating OpenShift behavior?
- Should the new cause have an `ErrorOnEviction`-style fail-fast opt-out?
