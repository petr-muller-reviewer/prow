---
pr: kubernetes-sigs/prow#931
title: "chore(deps): bump sigs.k8s.io/controller-runtime from 0.24.1 to 0.25.0 in the kubernetes group across 1 directory"
head_sha: 450cf57da53d1d3412fcde76945f4f45cc63dea1
base: main
reviewed_at: 2026-09-14T21:46:38Z
verdict: approve
---

## Verdict

Approve. This dependency-only update aligns controller-runtime with Prow's existing Kubernetes v0.37 direct dependencies. The release is 11 days old and has broad production use here, but its breaking and new behavior does not intersect with Prow's imported APIs. Merge after the pending normal CI completes.

## What this PR does

- Updates the direct controller-runtime dependency from v0.24.1 to v0.25.0.
- Updates its indirect k8s.io/apiextensions-apiserver dependency from v0.36.3 to v0.37.0.
- Refreshes the corresponding module checksums only.
- Does not modify Prow source, configuration, or tests.

## Findings

None.

## Checked

- `go.mod:83` declares controller-runtime directly; it is imported by 55 Go files (30 production and 25 test files), covering ProwJob reconcilers, managers, caches, clients, and fake-client tests.
- `go.mod:166` now aligns the transitive apiextensions-apiserver version with Prow's existing direct Kubernetes v0.37.0 dependencies.
- controller-runtime v0.25.0 is a tagged release published 2026-09-03 (11 days before review); the v0.37.0 Kubernetes release was published 2026-08-26.
- Upstream changes include opt-in read-your-writes cache consistency, fake-client enhancements and fixes, controller/cache shutdown fixes, and the Kubernetes v1.37 dependency update. Prow does not use the new read-your-writes, typed informer, EventRecorder, PriorityQueue, or affected fake-client APIs.
- GitHub's published security-advisory endpoint returned no controller-runtime advisory. The PR changes only `go.mod` and `go.sum`.

## Open questions

- Please merge after the currently pending required CI checks complete.
