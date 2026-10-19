---
pr: kubernetes-sigs/prow#873
title: "chore(deps): bump the kubernetes group across 2 directories with 4 updates"
head_sha: c6ccf0f187b7d8a706e2fba311b78f0cb58f65a9
base: main
reviewed_at: 2026-09-01T18:48:09Z
verdict: approve
---

## Verdict

Approve. This is a dependency-only Kubernetes v0.36 patch release update. The releases had 12 days of soak time when reviewed, the only behavioral client-go fix is not used by Prow, and the PR's unit, integration, image-build, and lint checks passed.

## What this PR does

- Updates root production dependencies `k8s.io/api`, `k8s.io/apimachinery`, and `k8s.io/client-go` from v0.36.3 to v0.36.4.
- Updates the `hack/tools` code-generation dependency `k8s.io/code-generator` from v0.36.3 to v0.36.4.
- Regenerates the corresponding checksums and raises the tools module's indirect api/apimachinery selections to v0.36.4.

## Dependency analysis

- `k8s.io/api` (`go.mod:79`), direct; v0.36.3 -> v0.36.4. Tagged 2026-08-20, 12 days old. Its release change is a Go dependency refresh. Prow imports it in 288 Go files (123 tests), so it is broadly used for Kubernetes API types, but this patch contains no API-specific behavioral change. Safe to take.
- `k8s.io/apimachinery` (`go.mod:80`), direct; v0.36.3 -> v0.36.4. Tagged 2026-08-20, 12 days old. The patch refreshes Go dependencies. Prow imports it in 280 Go files (120 tests), including configuration and controller paths; no applicable behavior change was identified. Safe to take.
- `k8s.io/client-go` (`go.mod:81`), direct; v0.36.3 -> v0.36.4. Tagged 2026-08-20, 12 days old. The substantive client-go change restores `FakeCustomStore` conformance to `cache.Store`, alongside dependency refreshes. Prow has no `FakeCustomStore`/`CustomStore` use; its 76 imports (17 test files) do use real Kubernetes clients. Safe to take.
- `k8s.io/code-generator` (`hack/tools/go.mod:14`), direct tool-only; v0.36.3 -> v0.36.4. Tagged 2026-08-20, 12 days old. The patch is dependency refresh only. It is imported solely by `hack/tools/tools.go`; no runtime Prow exposure. Safe to take.

## Findings

None.

## Checked

- Classified as dep-only: only the two Go manifests and their sums changed.
- All four selections are real v0.36.4 tags, not pseudo-versions.
- `go mod verify` passed in the root module and `hack/tools` module.
- GitHub checks passed: unit, integration, image build, lint, EasyCLA, and Tide.

## Open questions

None.
