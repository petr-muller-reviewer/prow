---
pr: kubernetes-sigs/prow#1001
title: "docs: generate the metrics reference from Go declarations"
head_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
base: main
reviewed_at: 2026-10-09T15:27:50Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-10-10T16:13:29Z
  gated_head_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
  reviewed_head_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
---

# Review

## Gate

**Decision: do-not-merge**

The prior blocking finding remains unresolved at the current PR head, which is unchanged from the reviewed head. The generator still accepts absent `Help` fields, and the generated descriptions for two metrics are blank. No submitted reviews or substantive inline comments were found; issue comments were bot notifications. The PR remains open.

### Gating list

- **Not addressed — REVIEW.md, “Require help text for every generated metric row”** (`hack/gen-prow-documented/metrics.go:298-312`): The parser assigns an empty description when `Help` is absent and only rejects an empty metric name. `jira_request_duration_seconds` and `prow_job_runtime_seconds` still have no `Help` in their declarations. Do not merge until both descriptions are supplied and the generator rejects absent or empty help.

### Independent merge risk

No notable merge risk. The five changed paths are limited to the metrics generator, its tests, the codegen verifier, and documentation. No exported runtime API, Prow configuration, or deployed behavior changes; the blast radius is limited to documentation generation and verification, so no operator migration or release note is needed.

## Verdict

Request changes. Code Quality and Maintainability reviewers independently confirmed that the generator accepts missing metric help and emits blank descriptions for two rows, contrary to the PR's stated requirement. Deployment risk is low because the change affects documentation tooling and generated docs, not deployed Prow behavior.

## What this PR does

- Parses Prometheus declarations under `cmd/` and `pkg/` to build a metrics reference.
- Resolves static names, labels, and help values, then sorts and links rows to their source definitions.
- Preserves surrounding guidance and documents HTTP helper and exporter metrics separately.
- Adds the metrics page to code generation and verification.

## Findings

### [blocking] Require help text for every generated metric row
- where: `hack/gen-prow-documented/metrics.go:298-312`
- concern: When `Help` is absent, the parser assigns an empty string to the row and only rejects an empty `Name`. The generated descriptions for `jira_request_duration_seconds` and `prow_job_runtime_seconds` are blank. Add help to those source declarations and reject missing or empty help in the generator so future declarations cannot silently produce incomplete documentation.
- excerpt: |
    for _, field := range []string{"Namespace", "Subsystem", "Name", "Help"} {
        value := ""
        if expression, ok := fields[field]; ok {
            value, err = stringValue(expression, definitions, map[ast.Expr]bool{})
            if err != nil {
                return row, fmt.Errorf("%s: %w", field, err)
            }
        }
        if field == "Help" {
            row.help = value
        } else if value != "" {
            parts = append(parts, value)
        }
        if field == "Name" && value == "" {
            return row, fmt.Errorf("empty metric name")
        }

## Checked

- The code quality and maintainability reviewers independently confirmed the missing-help issue; the code and generated rows were checked directly.
- Static aliases, constructor variants, runtime HTTP helper exclusions, sorting, source links, and codegen check integration were reviewed.
- Tests cover aliases, dynamic definitions, escaping, markers, and idempotent generation; tests were not run during this review.
- Deployment review found no runtime, configuration, dependency, or resource changes; deployment risk is low.
- No repository-specific code standards were found in `CONTRIBUTING.md`.

## Open questions

- Should the extractor's supported static declaration forms be documented so maintainers know how to extend it when new metric declaration patterns appear?
