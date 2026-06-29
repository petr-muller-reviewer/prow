---
pr: 117
title: "Revive prowjob when node is terminated (enabled by default)"
head_sha: b39206f58b33a1bd5ef9c348f87c263a5837789a
base: main
reviewed_at: "2026-06-29T15:13:51Z"
verdict: REQUEST_CHANGES
refresh_log:
  - previous_reviewed_at: "2026-04-29"
    refreshed_at: "2026-05-30T00:45:00Z"
    summary: "No code changes. Only bot activity: k8s-triage-robot lifecycle/stale comment on 2026-05-29."
  - previous_sha: b39206f58b33a1bd5ef9c348f87c263a5837789a
    refreshed_at: "2026-06-29T15:13:51Z"
    summary: "No code changes, no new comments or reviews. PR was closed without merging on 2026-06-28T21:35:52Z."
---

# PR #117: Revive prowjob when node is terminated (enabled by default)

**Author**: inteon (Tim Ramlot)
**URL**: https://github.com/kubernetes-sigs/prow/pull/117
**Size**: +171 / -19 | 9 files changed

## Final Recommendation: REQUEST_CHANGES

Two of three reviewers independently identified the same correctness gap in condition matching: checking `condition.Reason` without verifying `condition.Type == "DisruptionTarget"` or `condition.Status == corev1.ConditionTrue`. This deviates from how Kubernetes controllers are expected to consume Pod Disruption Conditions, and the configurable nature of `TerminationConditionReasons` amplifies the false-positive surface. Combined with zero test coverage for the condition-based detection path, this represents a real risk of silent incorrect behavior that would be hard to diagnose in production.

## Converging Concerns

Issues independently flagged by multiple reviewers:

### Condition matching lacks Type and Status checks

Flagged by: **Code Quality** + **Maintainability**

`pkg/plank/reconciler.go:693-696`

The condition-based path in `getPodUnexpectedStopCause` matches on `condition.Reason` alone. Per the Kubernetes Pod Disruption Conditions contract, it must also check `condition.Type == "DisruptionTarget"` and `condition.Status == corev1.ConditionTrue`. Without this, any condition of any type with a matching reason string would trigger revival.

```go
// Current (too broad):
for _, condition := range pod.Status.Conditions {
    if slices.Contains(terminationConditionReasons, condition.Reason) {
        return PodUnexpectedStopCauseTerminated
    }
}

// Recommended:
for _, condition := range pod.Status.Conditions {
    if condition.Type == "DisruptionTarget" &&
        condition.Status == corev1.ConditionTrue &&
        slices.Contains(terminationConditionReasons, condition.Reason) {
        return PodUnexpectedStopCauseTerminated
    }
}
```

### No test coverage for the condition-based detection path

Flagged by: **Code Quality** + **Maintainability**

All three new test cases use `pod.Status.Reason = "Terminated"`, leaving the `pod.Status.Conditions` matching path entirely unexercised. This is the primary mechanism for non-graceful shutdowns (PodGC, GCP controller), yet a regression would go undetected because the `Status.Reason` path would still work.

## Required Changes

1. **Tighten condition matching** (`pkg/plank/reconciler.go:693-696`): Add checks for `condition.Type == "DisruptionTarget"` and `condition.Status == corev1.ConditionTrue` when matching against `TerminationConditionReasons`.

2. **Add tests for the condition-based detection path** (`pkg/plank/controller_test.go`): At minimum: a positive test (condition with matching reason + correct Type/Status) and a negative test (reason matches but Type or Status does not).

## Individual Reviewer Assessments

### 1/3 — Code Quality (COMMENT)

- **Condition matching is too broad** (critical): `getPodUnexpectedStopCause` at `reconciler.go:693-696` matches `condition.Reason` alone — doesn't check `condition.Type` or `condition.Status`.
- **No tests for the condition-based detection path**: All new test cases use `pod.Status.Reason = "Terminated"`. No test exercises the `pod.Status.Conditions` matching logic.
- **No test for MaxRevivals limit after termination**: Only eviction/unreachable paths test the revival cap.
- **Empty vs. omitted `termination_condition_reasons`**: The `nil` check means `[]` disables condition matching while omitting gets defaults. Differs from the `MaxRevivals` pointer pattern and isn't documented.
- **Positive**: Clean adherence to `ErrorOnEviction` pattern, good finalizer cleanup, well-linked documentation comments, configurable reasons.

### 2/3 — Maintainability (COMMENT, LOW burden)

- **Condition-based path is untested** (convergent): If it breaks, the `Status.Reason` path would still work, making the regression hard to notice.
- **Missing `condition.Status` check** (convergent): Could cause false positives on pods with historical or inactive conditions.
- **Config can change mid-flight**: `TerminationConditionReasons` is read from live config on every reconciliation. Consistent with existing patterns but worth noting.
- **Test naming**: "delete terminated pod" doesn't convey that the pod will be revived. Better: "revive terminated pod by deleting it for recreation".
- **Suggestion**: Unit-test `getPodUnexpectedStopCause` directly as a pure function.
- **Positive**: Follows `ErrorOnEviction` pattern across all layers, good inline comments with K8s doc links, CRD/config/API types all in sync.

### 3/3 — Deployment Risk (MEDIUM risk)

- **Silent behavioral change on upgrade**: Terminated pods are now revived instead of failed. Activates without operator action. Alerts/workflows triggered by terminated-pod failures will stop firing.
- **Opt-in vs. opt-on-by-default**: Multiple project members (droslean, BenTheElder) raised this concern in the PR discussion. Consider making revival opt-in.
- **No way to fully disable revival via config**: `termination_condition_reasons: []` disables condition matching, but `Status.Reason == "Terminated"` still triggers revival. Only per-job `error_on_termination: true` is a full opt-out.
- **GCP-specific defaults**: `DeletionByGCPControllerManager` is GCP-specific. AWS/Azure/bare-metal may use different condition reasons.
- **CRD ordering**: Must be applied before deploying the new plank binary (additive, safe).
- **Mitigations**: `MaxRevivals` (default 3) prevents infinite loops, `ErrorOnTermination` per-job opt-out exists, finalizer cleanup handled, rollback is safe.

## Non-blocking Suggestions

- Unit-test `getPodUnexpectedStopCause` directly as a pure function — covers both `Status.Reason` and `Status.Conditions` paths cleanly.
- Add a test case verifying behavior when `MaxRevivals` is reached through the termination path specifically.
- Document the semantic difference between `TerminationConditionReasons: nil` (uses defaults) and `TerminationConditionReasons: []` (no condition-based matching), or normalize so empty also falls back to defaults.
- Consider making `Status.Reason == "Terminated"` revival also configurable/disableable for operators who want full control.
- Rename test case "delete terminated pod" to convey revival intent.

## Since Previous Review (2026-05-30 → 2026-06-29)

- **PR closed without merging** on 2026-06-28T21:35:52Z. No new commits, no new comments or reviews.
- No code changes — the two required findings (condition-matching correctness, missing tests) remain unaddressed.
- Likely abandoned rather than actively being worked; the required changes were not made.

## Deployment Notes

- Terminated pods are now revived instead of failed on upgrade — no operator action required to activate.
- Operators relying on terminated-pod failures for alerts/automation will see those stop firing silently.
- CRD must be updated before deploying the new plank binary.
- Per-job opt-out: `error_on_termination: true`. Global cap: `max_revivals` (default 3).
- Rollback is safe — no data migration involved.
