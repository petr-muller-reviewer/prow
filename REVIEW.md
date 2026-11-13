---
pr: kubernetes-sigs/prow#946
title: "chore(deps): bump sigs.k8s.io/controller-runtime from 0.25.0 to 0.25.1 in the kubernetes group across 1 directory"
head_sha: aebcb76d0da356a4f520cfe692738010b2a14e00
base: main
reviewed_at: 2026-11-13T17:21:58Z
verdict: approve
refresh_log:
  - old_sha: aebcb76d0da356a4f520cfe692738010b2a14e00
    new_sha: aebcb76d0da356a4f520cfe692738010b2a14e00
    at: 2026-11-13T17:21:58Z
    summary: No code changes after the review; PR was approved, merged, and closed with the reviewed head unchanged.
---

## Verdict

Approve. This is a direct, dependency-only patch. controller-runtime v0.25.1 is a tagged release published 8–9 days before review; its two fixes do not alter an API path Prow uses, and the remaining fix removes an upstream data race.

## What this PR does

- Updates the direct Go dependency `sigs.k8s.io/controller-runtime` from `v0.25.0` to `v0.25.1`.
- Refreshes the corresponding module checksums in `go.sum`.
- Takes an upstream fix for a priority-queue logging data race.
- Takes an upstream fix for cached-client subresource creation with read-your-writes consistency enabled.

Since previous review (2026-11-13):

- No code changes: head is still `aebcb76d0da356a4f520cfe692738010b2a14e00`, the commit reviewed on 2026-09-22.
- PR was approved by petr-muller at 2026-09-22T21:53:12Z (the published form of this review), APPROVALNOTIFIER confirmed at 21:53:19Z, and the PR was merged into `main` at 2026-09-22T22:22:11Z — the verdict above stands as the record of the merged change.

## Findings

No findings.

## Checked

- Classification: dep-only; the diff is limited to `go.mod` and `go.sum`.
- Provenance and freshness: v0.25.1 is a non-prerelease tag (`67b72c2517be1d2b0dec612477eb20c3c959a8aa`), published 2026-09-14; no upstream repository security advisory was reported for this release.
- Upstream delta: two commits only—priority-queue logging race fix and the cached-client subresource-create resource-version parsing fix.
- Usage: Prow imports controller-runtime in 55 Go files (30 production files), across controller managers and Kubernetes clients. It has no `SubResource` calls and does not enable `EnableReadYourWritesConsistency`, so the changed cached-client behavior is not exercised here.
- Validation: `git diff --check` and `go mod verify` pass. PR CI reports successful unit, integration, image-build, lint, and nonblocking race checks.

## Open questions

None.

## Dependency followups

No improvement opportunities identified.

- Modules examined: `sigs.k8s.io/controller-runtime` `v0.25.0 → v0.25.1` (direct). No indirect modules changed — the diff is `go.mod` (one line) plus two `go.sum` lines.
- Changelog range (v0.25.0...v0.25.1, 2 commits): priority-queue `logState` data-race fix and the cached-client subresource-create resource-version fix. Neither deprecates an API Prow calls, adds an API Prow could adopt, nor changes a default Prow overrides.
- Considered and rejected: adopting `CacheOptions.EnableReadYourWritesConsistency` to drop the hand-rolled "wait until the write reaches the cache" poll loops at `pkg/plank/reconciler.go:659`, `pkg/plank/reconciler.go:775`, `pkg/crier/reporters/criercommonlib/updatereportstatus.go:52` (and the 10s variant at `pkg/plank/reconciler.go:882`). The option already exists in v0.25.0, so this bump does not unlock it; v0.25.1 only fixes the option's `SubResource("eviction").Create` path, and Prow has no controller-runtime subresource calls (the only `SubResource(...)` use is `test/integration/test/setup.go:219`, a plain clientset exec subresource).
