---
pr: kubernetes-sigs/prow#1001
title: "docs: generate the metrics reference from Go declarations"
head_sha: 8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b
base: main
reviewed_at: 2026-10-21T14:00:32Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-10-15T23:17:51Z
  gated_head_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
  reviewed_head_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
refresh_log:
  - old_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
    new_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
    at: 2026-10-15T23:14:23Z
    summary: "No code changes; kubernetes-prow[bot] commented that the PR needs rebase."
  - old_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
    new_sha: 8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b
    at: 2026-10-21T14:00:32Z
    summary: "PR branch updated onto main; generator unchanged and the missing-help finding remains."
---

# Review

## Gate

**Decision: do-not-merge**

The prior gate was recorded at `7a209d855110d9c9a13b66c418a93d2cdd29c75e`. The PR head has since advanced to `8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b` after updating onto `main`; the generator implementation is unchanged and both blank descriptions remain. This refresh did not rerun the gate. The PR remains open, with no new submitted reviews or comments since the previous review.

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

**Since previous review:**

- The PR head advanced from `7a209d855110d9c9a13b66c418a93d2cdd29c75e` to `8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b` while updating onto `main`. The old-to-new history spans 36 commits, but only the existing feature commit is ahead of current `main`; the current PR diff remains the same five files, with no changes to the generator implementation.
- The generated documentation has updated source line links for `cmd/sinker`; the two empty descriptions remain. No new reviews, inline comments, or issue comments were added, and the PR remains open.

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
