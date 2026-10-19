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

## Dependency followups

Examined 2026-10-19. No actionable opportunities identified; no items or handoff prompts.

PR #873 merged on 2026-08-25 as `2435c489ae376aa4e36eb567890c90682abb7e57`. The ranges below are taken from PR head `c6ccf0f187b7d8a706e2fba311b78f0cb58f65a9` against its parent; the current local HEAD additionally contains the saved-review commit, and the PR is already an ancestor of the default branch.

- Direct root modules `k8s.io/api`, `k8s.io/apimachinery`, and `k8s.io/client-go`: v0.36.3 → v0.36.4. Direct tools module `k8s.io/code-generator`: v0.36.3 → v0.36.4.
- Transitive root module `k8s.io/streaming`: v0.36.3 → v0.36.4. Tools-module indirect selections: `k8s.io/api` v0.36.0 → v0.36.4 and `k8s.io/apimachinery` v0.36.3 → v0.36.4. No other module versions change in the manifests or checksums.
- The [client-go comparison](https://github.com/kubernetes/client-go/compare/v0.36.3...v0.36.4) adds `BookmarkFunc` and `LastStoreSyncResourceVersionFunc` to `FakeCustomStore`, restoring its `cache.Store` conformance. Prow has no `FakeCustomStore` or `CustomStore` references, so this does not replace any implementation or workaround we own.
- The [api patch comparison](https://github.com/kubernetes/api/compare/v0.36.3...v0.36.4), [apimachinery comparison](https://github.com/kubernetes/apimachinery/compare/v0.36.3...v0.36.4), [code-generator comparison](https://github.com/kubernetes/code-generator/compare/v0.36.3...v0.36.4), and [streaming comparison](https://github.com/kubernetes/streaming/compare/v0.36.3...v0.36.4) change dependency files only. They introduce no APIs, deprecations, or defaults requiring Prow call-site changes. Prow has no direct streaming imports; its generator imports are in `hack/tools/tools.go:28-31`.
- The wider [tools api range](https://github.com/kubernetes/api/compare/v0.36.0...v0.36.4), including intermediate releases, also changes dependency files only. It includes a structured-merge-diff regression correction, without a new API to adopt. The PR does not change Prow's selected structured-merge-diff version.
- Upstream refreshes of `golang.org/x/net`, `text`, `sys`, `term`, `mod`, `sync`, and `tools` do not raise Prow's selections: both manifests already select newer versions than those required by these Kubernetes patches. Their features therefore are not unlocked by this PR.
