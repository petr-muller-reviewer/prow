---
pr: kubernetes-sigs/prow#948
title: "chore(deps): bump github.com/tektoncd/pipeline from 1.15.1 to 1.16.0"
head_sha: 21567294b16610240634433186a9af2e621c8afe
base: main
reviewed_at: 2026-09-22T21:15:37Z
verdict: approve
---

## Verdict

approve — dependency-only update with clean focused compilation/tests. Tekton v1.16.0 is 22 days old; its one relevant behavioral change is enabled security contexts for Tekton-injected containers, which warrants normal rollout validation but is not a defect in this PR.

## What this PR does

- Updates direct dependency `github.com/tektoncd/pipeline` from `v1.15.1` to `v1.16.0`.
- Updates its transitive dependency `github.com/google/cel-go` from `v0.29.2` to `v0.31.0`.
- Refreshes the associated module checksums.
- Does not alter Prow source, generated clients, configuration, or tests.

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
