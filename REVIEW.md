---
pr: kubernetes-sigs/prow#948
title: "chore(deps): bump github.com/tektoncd/pipeline from 1.15.1 to 1.16.0"
head_sha: 21567294b16610240634433186a9af2e621c8afe
base: main
reviewed_at: 2026-11-13T17:20:21Z
verdict: approve
refresh_log:
  - at: 2026-11-13T17:20:21Z
    old_sha: 21567294b16610240634433186a9af2e621c8afe
    new_sha: 21567294b16610240634433186a9af2e621c8afe
    summary: "No code changes; petr-muller approved the PR and it was merged."
---

## Verdict

approve — dependency-only update with clean focused compilation/tests. Tekton v1.16.0 is 22 days old; its one relevant behavioral change is enabled security contexts for Tekton-injected containers, which warrants normal rollout validation but is not a defect in this PR.

## What this PR does

- Updates direct dependency `github.com/tektoncd/pipeline` from `v1.15.1` to `v1.16.0`.
- Updates its transitive dependency `github.com/google/cel-go` from `v0.29.2` to `v0.31.0`.
- Refreshes the associated module checksums.
- Does not alter Prow source, generated clients, configuration, or tests.

Since previous review:
- No new commits; the PR head is unchanged at `21567294b16610240634433186a9af2e621c8afe`.
- `petr-muller` submitted an `APPROVED` review on 2026-09-22T21:35:38Z, followed by the approval-notifier bot comment at 21:35:45Z.
- The PR was merged on 2026-09-22T21:56:21Z.

## Findings

No findings.

## Checked

- PR scope is limited to `go.mod` and `go.sum`; the diff is whitespace-clean.
- `github.com/tektoncd/pipeline` is a tagged release from 2026-08-31 (22 days old), directly imported by Prow's Pipeline controller, config/API types, and generated Pipeline clients.
- Tekton v1.16.0 enables `set-security-context` by default for Tekton-injected TaskRun containers and Affinity Assistants. Prow's direct usage is API/client based; user-defined task steps and sidecars are outside that default's scope.
- `github.com/google/cel-go` is an indirect dependency with no direct Prow imports. v0.31.0 is a tagged release from 2026-08-07 (46 days old); its release notes report no breaking changes and add regex expression-size limits.
- `go test ./cmd/pipeline ./pkg/config ./pkg/apis/prowjobs/v1 ./pkg/pipeline/...` passed.

## Open questions

None for this dependency update. Operators should exercise representative Tekton PipelineRuns during rollout if their injected containers depend on permissive security contexts.

## Dependency followups

No opportunities identified. Examined:

- `github.com/tektoncd/pipeline` `v1.15.1` → `v1.16.0` (direct). The only Tekton package Prow imports is `github.com/tektoncd/pipeline/pkg/apis/pipeline/v1` (e.g. `cmd/pipeline/controller.go:40`, `pkg/apis/prowjobs/v1/types.go:29`, `pkg/config/config.go:47`, and the generated `pkg/pipeline/` clients); that package is unchanged between the two tags. The v1.16.0 source changes are internal reconciler/tracing code, the `set-security-context` default flip in `pkg/apis/config/feature_flags.go` (a package Prow does not import), and dependency/CI churn. No deprecations or new exported APIs reach Prow call sites.
- `github.com/google/cel-go` `v0.29.2` → `v0.31.0` (indirect). Prow has no imports of `cel-go`; the bump is transitive through Tekton, and Tekton's v1.16.0 notes only list the version bump, so there is no Prow call site to migrate or simplify.
