---
pr: kubernetes-sigs/prow#946
title: "chore(deps): bump sigs.k8s.io/controller-runtime from 0.25.0 to 0.25.1 in the kubernetes group across 1 directory"
head_sha: aebcb76d0da356a4f520cfe692738010b2a14e00
base: main
reviewed_at: 2026-09-22T21:33:31Z
verdict: approve
---

## Verdict

Approve. This is a direct, dependency-only patch. controller-runtime v0.25.1 is a tagged release published 8–9 days before review; its two fixes do not alter an API path Prow uses, and the remaining fix removes an upstream data race.

## What this PR does

- Updates the direct Go dependency `sigs.k8s.io/controller-runtime` from `v0.25.0` to `v0.25.1`.
- Refreshes the corresponding module checksums in `go.sum`.
- Takes an upstream fix for a priority-queue logging data race.
- Takes an upstream fix for cached-client subresource creation with read-your-writes consistency enabled.

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
