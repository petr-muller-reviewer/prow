---
pr: kubernetes-sigs/prow#1001
title: "docs: generate the metrics reference from Go declarations"
head_sha: 8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b
base: main
reviewed_at: 2026-11-11T11:47:49Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-11-11T11:17:20Z
  gated_head_sha: 8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b
  reviewed_head_sha: 8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b
refresh_log:
  - old_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
    new_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
    at: 2026-10-15T23:14:23Z
    summary: "No code changes; kubernetes-prow[bot] commented that the PR needs rebase."
  - old_sha: 7a209d855110d9c9a13b66c418a93d2cdd29c75e
    new_sha: 8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b
    at: 2026-10-21T14:00:32Z
    summary: "PR branch updated onto main; generator unchanged and the missing-help finding remains."
  - old_sha: 8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b
    new_sha: 8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b
    at: 2026-11-11T11:47:49Z
    summary: "No code changes; incorporated petr-muller's help-text comment and recorded documentation losses identified in the local follow-up comparison."
---

# Review

## Gate

**Decision: do-not-merge**

The gate at `2026-11-11T11:17:20Z` covered `8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b`, which remains the current head. Direct inspection confirmed that the blocking finding, “Require help text for every generated metric row,” was unaddressed: the generator accepts absent or empty help and the two documented rows have blank descriptions. This prevents the reference from meeting the PR's stated requirement that each row includes help text. There were no submitted reviews, inline comments, or human issue comments at that gate. Since then, @petr-muller requested the same help-text check and submitted a COMMENTED review. This refresh also records documentation losses identified during the local follow-up comparison; the gate has not been rerun against those additional findings.

### Gating list

- **Not addressed — REVIEW.md, “Require help text for every generated metric row”** (`hack/gen-prow-documented/metrics.go:298-312`): The parser defaults missing `Help` to an empty string and only rejects an empty metric name. The declarations at `pkg/jira/metrics.go:33` and `pkg/metrics/prowjobs/collector.go:118` still omit `Help`; their generated descriptions at `site/content/en/docs/metrics/_index.md:106` and `site/content/en/docs/metrics/_index.md:109` are blank. **Blocks merge:** add both descriptions and reject absent or empty help in the generator.

### Independent merge risk

No notable merge risk beyond the unresolved finding. The full PR changes five files (991 insertions, 59 deletions), all in the documentation generator, its tests, the codegen verifier, and the metrics reference. There are no exported runtime API, CRD, deployment configuration, or deployed behavior changes. The generator now rewrites the metrics table and exits with an error when extraction fails (`hack/gen-prow-documented/main.go:123`); the verifier also checks this page (`hack/make-rules/verify/codegen.sh:25`). The blast radius is maintainers running documentation generation and verification; existing Prow installations need no migration or coordinated rollout. No applicable repository compatibility skill was available; this pass used direct code inspection. Tests were not run for this gate.

## Verdict

Request changes. Code Quality and Maintainability reviewers independently confirmed that the generator accepts missing metric help and emits blank descriptions for two rows, contrary to the PR's stated requirement. A follow-up comparison also found that the replacement loses operator context from the handcrafted reference. Deployment risk is low because the change affects documentation tooling and generated docs, not deployed Prow behavior.

## What this PR does

- Parses Prometheus declarations under `cmd/` and `pkg/` to build a metrics reference.
- Resolves static names, labels, and help values, then sorts and links rows to their source definitions.
- Preserves surrounding guidance and documents HTTP helper and exporter metrics separately.
- Adds the metrics page to code generation and verification.

**Since previous review:**

- The PR head advanced from `7a209d855110d9c9a13b66c418a93d2cdd29c75e` to `8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b` while updating onto `main`. The old-to-new history spans 36 commits, but only the existing feature commit is ahead of current `main`; the current PR diff remains the same five files, with no changes to the generator implementation.
- The generated documentation has updated source line links for `cmd/sinker`; the two empty descriptions remain. No new reviews, inline comments, or issue comments were added, and the PR remains open.

**Since previous review (2026-11-11):**

- No code changes: the PR remains open at `8c27c2ddf45c23ccbd2e89c3382ef82c42cb0d0b`.
- @petr-muller added an inline comment at `2026-11-11T11:28:10Z` on `hack/gen-prow-documented/metrics.go:307`, requesting nonempty help validation and descriptions for the two exceptions, then submitted a COMMENTED review at `2026-11-11T11:33:18Z`. This reinforces the existing blocking finding; there are no new issue comments.
- The local follow-up comparison found lost Tide interpretation details, component ownership mappings, and static-job metric metadata. These are recorded below as a should-fix finding; the static-job operational guidance, replica warning, Pushgateway explanation, and monitoring link are preserved.

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

### [should-fix] Preserve operator context from the handcrafted reference
- where: `site/content/en/docs/metrics/_index.md:11`, `site/content/en/docs/metrics/_index.md:121`, `site/content/en/docs/metrics/_index.md:157`
- concern: Replacing the handcrafted table loses useful information: Tide descriptions no longer explain pagination, inclusion of partial results in PR counts and error counts, or the completeness ratio's latest nonempty search cycle; source-package links replace component mappings such as Hook and Plank/Jenkins-Operator. The `prow_config_static_job_definition` row also disappears because its `prometheus.NewDesc` declaration is unsupported by the extractor, losing its explicit Gauge type and `type` label even though its explanatory section survives. Preserve the richer Tide wording in Go Help, retain component mappings in surrounding guidance, and document the static-job metric's type and label outside the generated table or extend extraction to cover it.
- excerpt: |
    | [pkg/tide](https://github.com/kubernetes-sigs/prow/blob/main/pkg/tide/tide.go#L347) | Gauge | `tide_pool_completeness_ratio` | controller | Fraction of query shards that completed fully. |
    | [pkg/tide](https://github.com/kubernetes-sigs/prow/blob/main/pkg/tide/tide.go#L308) | Histogram | `tide_query_duration_seconds` | controller, result | Duration of individual Tide GitHub search queries per shard. |
    | [pkg/tide](https://github.com/kubernetes-sigs/prow/blob/main/pkg/tide/tide.go#L316) | Histogram | `tide_query_prs_returned` | controller | Number of PRs returned per Tide query shard. |

## Checked

- The code quality and maintainability reviewers independently confirmed the missing-help issue; the code and generated rows were checked directly.
- Static aliases, constructor variants, runtime HTTP helper exclusions, sorting, source links, and codegen check integration were reviewed.
- Tests cover aliases, dynamic definitions, escaping, markers, and idempotent generation; tests were not run during this review.
- Deployment review found no runtime, configuration, dependency, or resource changes; deployment risk is low.
- No repository-specific code standards were found in `CONTRIBUTING.md`.
- Compared the handcrafted metrics page at `8c27c2d^` with `8c27c2d`: the main table expands from 48 unique metric names to 105, but drops the static-job metric row and some operator context. This comparison used static inspection; no tests were run.

## Open questions

- Should the extractor's supported static declaration forms be documented so maintainers know how to extend it when new metric declaration patterns appear?
