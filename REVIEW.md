---
pr: kubernetes-sigs/prow#1061
title: "chore(deps): bump github.com/prometheus/common from 0.73.0 to 0.73.1 in the prometheus group across 1 directory"
head_sha: 0bf29ff66ccc3728c91b83023a5bd53e0ff981b6
base: main
reviewed_at: 2026-11-07T21:34:13Z
verdict: approve
---

# Review

## Verdict

Approve — safe, low-risk dependency bump.

The PR bumps `github.com/prometheus/common` 0.73.0 → 0.73.1, whose only upstream change is a transitive bump of `github.com/pandatix/go-abnf` 0.5.1 → 0.5.3. go-abnf's only library change swaps `hashicorp/go-uuid` for `crypto/rand`-backed node IDs inside its grammar transition graph — a path prow never calls. Prow uses prometheus/common only in `pkg/metrics/push.go` (the `expfmt` encoder and `model` label validation), there are no OSV advisories at either version, and the govulncheck reachable set is identical before and after. Both modules are tidy, the build and module verification pass, and CI is green.

## What this PR does

- Bumps direct dependency `github.com/prometheus/common` 0.73.0 → 0.73.1 in the main module, and the same version in `hack/tools` where it is indirect.
- Lifts transitive `github.com/pandatix/go-abnf` 0.5.1 → 0.5.3, which is the sole change in prometheus/common 0.73.1 (prometheus/common#1027).
- Drops the now-unneeded `github.com/hashicorp/go-uuid v1.0.3` indirect requirement from both modules, because go-abnf stopped using it.
- Touches only `go.mod`, `go.sum`, `hack/tools/go.mod`, and `hack/tools/go.sum`; no project source, config, docs, or tests change.

## Findings

None.

## Resolved

None.

## Checked

- Dep-only: `git diff --stat $BASE..HEAD` shows exactly the four manifest/lock files; there is no `vendor/` directory in the repo.
- `prometheus/common` v0.73.1 (released 2026-10-30T09:45Z) contains a single commit — bump `go-abnf` to 0.5.3 — with no changes to prometheus/common's own code.
- `go-abnf` 0.5.1 → 0.5.3 changed files are CI workflows, tool module files, and `transition_graph.go`; the only library change replaces `hashicorp/go-uuid` with `crypto/rand` 16-byte hex IDs. Node IDs only need per-graph uniqueness, so behavior is equivalent.
- Import surface: only `pkg/metrics/push.go:31-32` imports prometheus/common (`expfmt`, `model`). `go mod why` confirms the single chain `sigs.k8s.io/prow/pkg/metrics → github.com/prometheus/common/expfmt`. Prow does not import go-abnf directly; it is reached only through expfmt's OpenMetrics parser, which `push.go` does not call (it encodes, not parses).
- OSV: no advisories for prometheus/common 0.73.0 or 0.73.1, and none for go-abnf 0.5.1 or 0.5.3.
- govulncheck v1.9.0 module-mode scan cannot run on this repo root (`no Go files in /workspace`); symbol-level scan scoped to `./pkg/metrics/...` at base and head produced identical advisory sets (GO-2023-1901, GO-2026-5932, GO-2026-6860, GO-2026-6866), none involving either bumped module.
- `go build ./pkg/metrics/...` passes; `go mod verify` reports all modules verified; `go mod tidy -diff` is clean in both the root module and `hack/tools`.
- Go directives: repo `go 1.26.4`, `hack/tools` `go 1.26.3`; both bumped deps declare `go 1.26.0`.
- CI: unit, integration, lint, image-build, and race-detector jobs pass; `tide` is pending only on `approved`/`lgtm` labels.

## Open questions

None.

## Dependency followups

No followup opportunities identified.

- `github.com/prometheus/common` 0.73.0 → 0.73.1 — the release is only a `go-abnf` bump; it adds no deprecations, new APIs, or behavior changes affecting the `expfmt`/`model` calls in `pkg/metrics/push.go`.
- `github.com/pandatix/go-abnf` 0.5.1 → 0.5.3 — internal node-ID generation swap (`hashicorp/go-uuid` → `crypto/rand`), CI/dependency churn, and a `go` directive lowering; no public API changes, and prow does not import the module directly.
