---
pr: kubernetes-sigs/prow#747
title: "Update base container images to v20260512-1f34ef0df3"
head_sha: 6312f82b3a02dd5f49811f354419ab351fbec409
base: main
reviewed_at: 2026-06-12T10:05:36Z
verdict: approve
---

## Summary

Routine bump of all `gcr.io/k8s-staging-test-infra/{alpine,git,git-custom-k8s-auth}` base images in `.ko.yaml` from `v20251206-8481528a19` to `v20260512-1f34ef0df3`. Single file, no logic changes. Three independent reviewers (code quality, maintainability, deployment risk) all returned APPROVE with no critical issues.

## Findings

### [nit] No PR description
- where: `.ko.yaml` (PR metadata)
- concern: PR body is empty. A one-liner referencing the upstream image promotion run or commit would make provenance auditable without extra investigation.
- excerpt: |
    (no PR body)

### [question] Image pullability pre-merge check
- where: `gcr.io/k8s-staging-test-infra` registry
- concern: Deployment risk reviewer flagged that the new tags should be confirmed pullable before merge to avoid a broken release build. This is a staging registry; availability is not guaranteed.
- excerpt: |
    gcr.io/k8s-staging-test-infra/alpine:v20260512-1f34ef0df3
    gcr.io/k8s-staging-test-infra/git:v20260512-1f34ef0df3
    gcr.io/k8s-staging-test-infra/git-custom-k8s-auth:v20260512-1f34ef0df3

## Checked

- All 35 `gcr.io/k8s-staging-test-infra` entries consistently bumped; no entries missed.
- Image-to-base-type assignments (alpine / git / git-custom-k8s-auth) are unchanged — no component accidentally received a different image class.
- `webhook-server` (`gcr.io/distroless/static:nonroot@sha256:…`) left untouched — correct, digest-pinned separately.
- `fakepubsub` (`google/cloud-sdk:389.0.0`) left untouched — correct, versioned separately.
- New tag `v20260512-1f34ef0df3` follows the established `v<YYYYMMDD>-<sha>` convention.
- ~5 month cadence (Dec 2025 → May 2026) is normal for this type of bump.
- No configuration schema, API, ProwJob spec, or operator-facing behavior altered.
- No version skew: all three image families advance to the same tag atomically.
- Change is consistent with prior bump pattern (`da4351c35`).

## Open questions

- Can you confirm the new image tags are pullable from `gcr.io/k8s-staging-test-infra` before this merges?
- Is there an upstream image promotion PR or release link that could be added to the PR description for traceability?
