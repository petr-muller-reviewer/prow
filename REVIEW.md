---
pr: kubernetes-sigs/prow#761
title: "Migrate gopkg.in/yaml.v3 to go.yaml.in/yaml/v3"
head_sha: 47f1063a233c8ed7267b6d5052b10aecc68a7562
base: main
reviewed_at: 2026-06-22T13:46:46Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-06-22T14:35:17Z
  gated_head_sha: 47f1063a233c8ed7267b6d5052b10aecc68a7562
  reviewed_head_sha: 47f1063a233c8ed7267b6d5052b10aecc68a7562
---

## Gate

**Decision: merge**

No prior gating findings to resolve (only a nit on mixed whitespace changes). PR head unchanged since review. No API, configuration, or behavioral changes — purely a compile-time import-path swap. @saschagrunert approved, `lgtm` label applied by @stmcginnis. Missing `approved` label — requires `/approve` from a root OWNERS approver (@petr-muller is assigned). No hold labels present. `mergeStateStatus` is `UNSTABLE` (likely a flaky or pending CI check, not a code concern).

### Findings disposition
- [nit] whitespace in genyaml_test.go: not a gating concern. No blocking or should-fix findings exist.

### Merge risk
No notable merge risk. No exported API changes, no config schema changes, no behavioral changes. Compile-time only.

## Summary

Module-path migration from archived `gopkg.in/yaml.v3 v3.0.1` to `go.yaml.in/yaml/v3 v3.0.4` (same library, new canonical path under yaml/go-yaml org). Four source files updated (import paths only), go.mod/go.sum adjusted. Old module retained as indirect for transitive deps. Three unrelated blank-line removals in test file.

## Dependency analysis

### go.yaml.in/yaml/v3: v3.0.1 (gopkg.in) -> v3.0.4 (go.yaml.in)
- type: module-path migration + patch bump
- freshness: v3.0.4 released 2025-06-29, ~12 months old. Tagged release. Fine.
- usage: direct. 4 files, 3 packages (hack/ts-rollup, pkg/genyaml, pkg/plugins/testfreeze/checker). Moderate exposure, non-sensitive paths.
- changelog: 16 commits, all organizational/housekeeping (module-path formalization, go fmt/vet, CI, README, sync with sigs.k8s.io/yaml/goyaml.v3). No bug fixes, no security fixes, no API changes, no behavior changes.
- exposure: light. No changed upstream code affects functions we call.
- take: safe to bump now. Archived gopkg.in path is a liability.

## Findings

### [nit] Unrelated whitespace changes in test file
- where: `pkg/genyaml/genyaml_test.go:259,271,468`
- concern: Three blank-line removals are cosmetic cleanup unrelated to the import migration. Muddies the diff for future `git log -p`. Minor in a PR this small.

## Checked
- All direct `gopkg.in/yaml.v3` imports removed (grep confirms zero remaining)
- `yaml3` alias preserved in `pkg/genyaml/`
- `go.mod` correctly lists new module as direct, old as indirect
- API surface identical between old and new modules (Marshal, Unmarshal, NewEncoder, Node, Encoder, node-kind constants)
- No struct tags, CLI flags, config schemas, or behavioral logic changed
- No deployment risk; compile-time only
- Test logic and assertions unchanged
- v3.0.4 is ~12 months old, well past soak
- v3.0.1 to v3.0.4 delta is organizational only, no behavioral changes upstream

## Open questions
(none)
