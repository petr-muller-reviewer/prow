---
pr: kubernetes-sigs/prow#856
title: "chore(deps): bump the kubernetes group across 1 directory with 2 updates"
head_sha: 443a28f9bd3b6d998415791eb7da096cac59eeb6
base: main
reviewed_at: 2026-08-17T10:28:42Z
verdict: approve
---

## What this PR does
- Dependabot "kubernetes group" bump confined to `hack/tools/go.mod`/`hack/tools/go.sum` (separate build-tooling module, not the main `sigs.k8s.io/prow` module).
- Direct bumps: `k8s.io/code-generator` v0.32.8 → v0.36.3, `sigs.k8s.io/controller-tools` v0.17.3 → v0.21.0.
- Incidental churn: `go` directive 1.25.5 → 1.26.0, `google.golang.org/protobuf` bumped to a pseudo-version, batch of transitive `k8s.io/*` bumps riding along (api, apiextensions-apiserver, apimachinery, klog/v2, kube-openapi, utils, structured-merge-diff/v6).
- No project source changes — dep-only PR, no code review performed.

## Dependency analysis

### k8s.io/code-generator v0.32.8 → v0.36.3
- Freshness: released 2026-07-23 (~25 days old at review time). Fine.
- Usage: direct dep, confined to `hack/tools/tools.go` (`client-gen`, `deepcopy-gen`, `informer-gen`, `lister-gen` tool-pins) invoked as standalone binaries from `hack/make-rules/update/codegen.sh`. Not linked into any runtime binary.
- Changelog: 4-minor-version jump, but commits are mostly `validation-gen`/`applyconfiguration-gen` feature work (cohort support, UUID format tag, openapi model-name accessors) — generators Prow doesn't use. No CVEs found.
- Exposure: light — worst case is codegen diff surfacing in a follow-up regen PR, not runtime risk.
- Take: safe to take.

### sigs.k8s.io/controller-tools v0.17.3 → v0.21.0
- Freshness: released 2026-05-06 (~3.5 months old). Fine.
- Usage: direct dep, same pattern — `controller-gen` pinned in `hack/tools/tools.go`, invoked via `codegen.sh` to produce CRD/deepcopy manifests. Not linked into runtime.
- Changelog: v0.18-v0.21 releases are themselves mostly `k8s.io/*` bumps (to v0.33, v0.34, etc.) plus generator maintenance. No security-relevant findings.
- Exposure: light — same "regenerates checked-in code" risk profile.
- Take: safe to take.

## Findings

(none)

## Checked
- Diff scope confirmed dep-only (`git diff --stat` against merge-base with `upstream/main`): only `hack/tools/go.mod`/`hack/tools/go.sum` touched.
- Import surface of both direct bumps verified via grep — confined to `hack/tools/tools.go` and `hack/make-rules/update/codegen.sh`, not imported by the main module or any runtime path.
- Release ages for both direct bumps checked against proxy.golang.org — neither is within the "too fresh" window.
- Changelogs for both direct bumps reviewed (commit range for code-generator, release notes for controller-tools) — no CVEs, no behavior changes affecting the generators Prow actually uses.

## Open questions
(none)
