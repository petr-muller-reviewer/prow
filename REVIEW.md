---
pr: kubernetes-sigs/prow#895
title: "chore(deps): bump github.com/stretchr/testify from 1.12.0 to 1.12.1"
head_sha: 5713cd97ec386b4afff355237a5863de5ac75032
base: main
reviewed_at: 2026-09-01T11:06:38Z
verdict: approve
---

## Verdict

Approve. Dependency-only update with no Prow source changes. Testify v1.12.1 switches its YAML implementation from the archived `gopkg.in/yaml.v3` module to the maintained `go.yaml.in/yaml/v3`; the accompanying YAML v3.0.5 update has no runtime parser changes.

## What this PR does

- Updates direct test dependency `github.com/stretchr/testify` from `v1.12.0` to `v1.12.1`.
- Moves its YAML dependency from `gopkg.in/yaml.v3` to `go.yaml.in/yaml/v3`.
- Updates Prow's direct `go.yaml.in/yaml/v3` requirement from `v3.0.4` to `v3.0.5`.
- Removes the no-longer-selected `gopkg.in/yaml.v3` module checksum.

## Findings

None.

## Checked

- PR diff is limited to `go.mod` and `go.sum`; no project code changed.
- `github.com/stretchr/testify` is directly imported by five test files and only for ordinary `assert` helpers; no Prow test invokes its YAML assertion path.
- Testify v1.12.1 (tagged 2026-08-17) contains the YAML module migration; no API or assertion-behavior change was found.
- `go.yaml.in/yaml/v3` v3.0.5 (tagged 2026-07-26) changes upstream tests, CI, and Go-doc comment formatting; no parser/runtime implementation change was found.
- No repository security advisories were returned for `stretchr/testify`.

## Open questions

None.

## Dependency followups

Examined on 2026-10-21 for PR #895, merged as `23ef88d0950d7c912a8bbe9df70707a4205a6cf2`. No actionable opportunities or regressions identified; no items or handoff prompts recorded.

- `github.com/stretchr/testify` `v1.12.0` → `v1.12.1` (direct, test-only): the [upstream diff](https://github.com/stretchr/testify/compare/v1.12.0...v1.12.1) only switches the YAML dependency and updates associated comments. It introduces no new assertion APIs, deprecations, or defaults to adopt. Prow imports `assert` in five test files and has no `YAMLEq` calls or custom Testify YAML implementation.
- `go.yaml.in/yaml/v3` `v3.0.4` → `v3.0.5` (direct, advanced by Testify's requirement): the [upstream changes](https://github.com/yaml/go-yaml/compare/v3.0.4...v3.0.5) concern upstream tests, CI, documentation formatting, and retractions of unusable older tags. They offer no API or runtime behavior changes to apply to Prow's `Unmarshal`, `Marshal`, `Node`, or `Encoder` usage in `pkg/genyaml/genyaml.go`, `hack/ts-rollup/main.go`, and `pkg/plugins/testfreeze/checker/checker.go`.
- `gopkg.in/yaml.v3` `v3.0.1` (indirect) is removed from the selected requirements, replaced by the maintained YAML module above. Prow has no direct imports of the old module, so there is no remaining import migration to hand off. No other module versions change in this PR.
