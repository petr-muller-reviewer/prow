---
pr: kubernetes-sigs/prow#1006
title: "plank: persist pod revival count before deleting the pod"
head_sha: 357b80eb511d6b0f932e5e77edc23cf0e56e74a1
base: main
reviewed_at: 2026-10-10T14:07:32Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-10-10T14:09:38Z
  gated_head_sha: 357b80eb511d6b0f932e5e77edc23cf0e56e74a1
  reviewed_head_sha: 357b80eb511d6b0f932e5e77edc23cf0e56e74a1
---

# Review

## Gate

**Decision: do-not-merge**

The PR head is unchanged since the review, so the confirmed blocking finding remains. The retry count can still be incremented more than once for the same stopped pod when no build ID can be recovered.

**Gating list**

- `REVIEW.md` blocking finding (`pkg/plank/reconciler.go:483-515`): `revivalAlreadyCounted` is always false for an empty build ID. The current code still writes an empty annotation, allowing repeated stale or terminating observations to consume multiple revival attempts. Add a stable fallback identity and test that path before merging.
- No substantive GitHub review submissions or inline comments add other gating findings.

**Independent merge risk**

- **API/configuration:** The diff adds an exported constant and a ProwJob annotation. This is an additive Go API change; there are no removed or changed exported APIs, CRD schema changes, configuration changes, RBAC changes, or migration requirements.
- **Behavior:** Every Prow installation running plank will now persist unexpected-stop counts and enforce its existing `max_revivals` setting. This applies by default to jobs with unexpected pod stops and requires no operator action; the PR does not include a release note. The empty-build-ID case can make affected jobs hit the limit early, which is the unresolved blocker above.

## Verdict

Request changes. The revival count is persisted before pod deletion, but the per-pod deduplication fails when `getPodBuildID` returns an empty string. Code Quality and Deployment Risk independently identified this confirmed edge case; Maintainability found no concerns.

## What this PR does

- Persists a ProwJob's revival count before deleting an unexpectedly stopped pod.
- Records the counted pod's build ID in a ProwJob annotation to deduplicate repeated observations.
- Guards `getPodBuildID` when the pod has no containers.

## Findings

### [blocking] Empty build IDs bypass pod deduplication
- where: `pkg/plank/reconciler.go:483-515`
- concern: If `getPodBuildID` returns an empty string, `revivalAlreadyCounted` remains false and the annotation is written as empty. Repeated observations of the same terminating or stale pod can therefore increment `PodRevivalCount` repeatedly and reach `max_revivals` early. Use a stable fallback identity such as the pod UID and cover the empty-ID case in a test.
- excerpt: |
    revivalAlreadyCounted := podBuildID != "" && pj.Annotations[kube.RevivedBuildIDAnnotation] == podBuildID
    ...
    pj.Annotations[kube.RevivedBuildIDAnnotation] = podBuildID

## Checked

- The count and annotation are patched before finalizer removal and pod deletion.
- The ProwJob CRD has no status subresource, so the ordinary object patch can persist status and metadata together.
- The change adds no configuration, RBAC, or CRD migration requirements.
- Maintainability review found the logic localized and documented; tests cover known-build-ID deduplication and the max-revival case.
- No tests were run during this review.

## Open questions

- Can plank use a stable pod identity, such as the pod UID, when no build ID is available?
