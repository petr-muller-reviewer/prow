---
pr: kubernetes-sigs/prow#1060
title: "chore(deps): bump the kubernetes group across 2 directories with 4 updates"
head_sha: 3337385f106f4ebe528aa248da5ab8b84fe494ed
base: main
reviewed_at: 2026-11-07T21:25:11Z
verdict: approve
---

# Review

## Verdict

Approved — a patch-level Kubernetes library bump whose modules contain no source changes; safe to take now.

All four bumped modules (`k8s.io/api`, `k8s.io/apimachinery`, `k8s.io/client-go`, `k8s.io/code-generator`) move 0.37.1 → 0.37.2, and the only file-level delta between those tags is `go.mod`/`go.sum`; no Go code changed in any of them. `k8s.io/streaming` (indirect) moves to a tag resolving to the same upstream commit as 0.37.1. There are no OSV advisories at either version and `govulncheck` module scans are clean on both sides of the bump, so the heavy import surface (`k8s.io/api` in 292 files, `k8s.io/apimachinery` in 283, `k8s.io/client-go` in 76) carries no behavioral risk here. The release is 10 days old (2026-10-28), i.e. under two weeks: noted, but with zero code delta there is no soak-time argument against taking it.

## What this PR does

- Bumps `k8s.io/api`, `k8s.io/apimachinery` and `k8s.io/client-go` from v0.37.1 to v0.37.2 in the root module.
- Bumps `k8s.io/code-generator` v0.37.1 → v0.37.2 in `hack/tools`, dragging its indirect `k8s.io/api`/`k8s.io/apimachinery` pins along.
- Moves the indirect `k8s.io/streaming` from v0.37.1 to v0.37.2 in the root module.
- Touches only `go.mod`/`go.sum` (root and `hack/tools`); no project code, tests, config or docs change.

## Findings

No findings.

## Checked

- Classification: dep-only — `git diff 9f4d1868e..3337385f1 --stat` is 4 files, all manifests.
- Freshness: tags cut 2026-10-28 (upstream `kubernetes/kubernetes` v1.37.2 published 2026-10-28T04:37Z), ~10 days before `reviewed_at`; tagged releases, not pseudo-versions.
- OSV queries for `k8s.io/api`, `k8s.io/apimachinery`, `k8s.io/client-go`, `k8s.io/code-generator`, `k8s.io/streaming` at both v0.37.1 and v0.37.2: no advisories.
- `govulncheck -scan module`: clean at the merge base, at PR head, and for the `hack/tools` module. (Module scans cannot run from the repo root because Prow's module root has no Go files and govulncheck v1.9.0 loads `.`; ran them against temp copies of each `go.mod`/`go.sum`.) No symbol-level scan needed since neither side had module-level findings.
- Module content: `kubernetes/api`, `kubernetes/apimachinery`, `kubernetes/client-go` and `kubernetes/code-generator` compares v0.37.1...v0.37.2 show only `go.mod`/`go.sum` changes; `kubernetes/streaming` compares `identical` and its two module zips unpack to byte-identical trees.
- Release notes (k8s v1.37.2): Go 1.26.10 rebuild, DRA `NoExecute` device taint eviction, Job recreation expectations, apiserver 16384 request-header values, kube-proxy `nf_conntrack_max`. These live in `kubernetes/kubernetes` / `k8s.io/apiserver`, not in the modules bumped here, so none of them alter what Prow links.
- Build: `go build ./...` in the root module passes; `go mod tidy -diff` is clean for both the root and `hack/tools` modules.
- Pre-existing, unchanged by this PR: `k8s.io/apiextensions-apiserver` stays at v0.37.0 while the rest moves to v0.37.2 (v0.37.2 does exist upstream; the v0.37.0 pin is transitive).

## Open questions

- The v1.37.2 release notes carry server-side fixes (`k8s.io/apiserver` header limits, DRA eviction). Prow does not depend on `k8s.io/apiserver`, so I read this as client-only; worth confirming you do not want a matching `k8s.io/apiserver` bump in this group.

## Dependency followups

No opportunities identified. Examined `k8s.io/api`, `k8s.io/apimachinery`, `k8s.io/client-go` and `k8s.io/code-generator` at v0.37.1 → v0.37.2 (root module plus `hack/tools`) and the transitive `k8s.io/streaming` v0.37.1 → v0.37.2. The tag-to-tag diffs for all four direct modules contain only `go.mod`/`go.sum` changes, and `k8s.io/streaming` resolves to the same upstream commit with byte-identical content, so the bump introduces no deprecations, replacement APIs, new features or default changes that could act on our call sites. The v1.37.2 release-note entries (DRA eviction, Job expectations, apiserver header limits, kube-proxy sysctl) land in `kubernetes/kubernetes` and `k8s.io/apiserver`, which Prow does not import from this module graph.
