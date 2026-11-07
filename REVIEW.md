---
pr: kubernetes-sigs/prow#835
title: "chore(deps): bump github.com/google/cel-go from 0.26.0 to 0.29.0"
head_sha: 72b0241741c3886387cd811f0080a7d7a112b5f2
base: main
reviewed_at: 2026-08-18T12:51:53Z
verdict: approve
---

## Summary

Dep-only dependabot PR. Only `go.mod`/`go.sum` changed, no project source.

- `github.com/google/cel-go` v0.26.0 -> v0.29.0 (indirect)
- `github.com/antlr4-go/antlr/v4` v4.13.0 -> v4.13.1 (indirect, transitive of cel-go)
- `github.com/stoewer/go-strcase` v1.3.0 removed (indirect, unrelated tidy pruning)

## Dependency analysis

### github.com/google/cel-go v0.26.0 -> v0.29.0
- freshness: v0.29.0 tagged 2026-07-02 (`fa16799e`, real tag), ~47 days old at review time. Fine, soaked.
- usage: indirect only. `go mod why -m github.com/google/cel-go` resolves as `sigs.k8s.io/prow/cmd/pipeline -> github.com/tektoncd/pipeline/pkg/apis/pipeline/v1 -> github.com/google/cel-go/cel`. Zero direct imports of cel-go in prow's own source outside vendor.
- changelog: fixes across v0.27.x-v0.29.0 include optional-unwrapping fix, `has()` unknown-propagation fix, expression-size-limit enforcement, timezone-minute validation, int32/uint32 map-key-narrowing guard, OOM guard in `ext/lists` `genRange()`. Correctness/robustness fixes in the CEL evaluator, no CVEs.
- exposure: light. Prow never constructs/evaluates CEL expressions itself; only exercised via vendored Tekton Pipeline API validation inside `cmd/pipeline`. Not a hot or sensitive path for prow's own code.
- take: safe to bump now.

### github.com/antlr4-go/antlr/v4 v4.13.0 -> v4.13.1
- freshness: released 2024-05-15, long soaked.
- incidental patch bump pulled in by cel-go's antlr dependency; not independently reviewed further.

### github.com/stoewer/go-strcase (removed)
- pure `go mod tidy` pruning, no longer required transitively. No action needed.

## Findings

(none — dep-only PR, no project code to review)

## Checked
- `git diff --stat` confirms only `go.mod`/`go.sum` changed
- `go mod why -m github.com/google/cel-go` for import chain
- proxy.golang.org release timestamps for both bumped modules
- dependabot-provided changelog/release notes for cel-go v0.27.0-v0.29.0

## Open questions
(none)
