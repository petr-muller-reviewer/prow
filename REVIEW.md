---
pr: 622
title: "docs: replace sinker placeholder page with real component docs"
author: abdallhMoukdad
head_sha: 6a695980c0159b63432660955379526ab55a697d
base: main
reviewed_at: "2026-06-15T11:18:19Z"
verdict: APPROVE WITH SUGGESTIONS
refresh_log:
  - previous_sha: 6a695980c0159b63432660955379526ab55a697d
    new_sha: 6a695980c0159b63432660955379526ab55a697d
    date: "2026-06-15T11:18:19Z"
    summary: "No code changes. PR labeled lifecycle/rotten by k8s-triage-robot on 2026-06-14."
---

# PR #622 — docs: replace sinker placeholder page with real component docs

**Author:** abdallhMoukdad · [github.com/kubernetes-sigs/prow/pull/622](https://github.com/kubernetes-sigs/prow/pull/622) · +85 / -2 · Fixes #495

## Reviewer Verdicts

| Reviewer | Verdict | Summary |
|----------|---------|---------|
| Code Quality | APPROVE | Documentation-only PR. No Go code, logic, or tests changed. Standard code quality criteria do not apply. |
| Maintainability | APPROVE | Maintenance burden: LOW. Eliminates documentation debt with accurate content. Shared flag duplication is the only forward-looking concern. |
| Deployment Risk | APPROVE | Risk level: LOW. Zero operational impact. No code, config schemas, or deployment manifests touched. |

## Accuracy Audit

Every factual claim in the PR was checked against the source. Results below.

### Config defaults (`sinker` block in `config.yaml`)

| Key | Documented default | Source default | Status |
|-----|--------------------|----------------|--------|
| `resync_period` | `1h` | `time.Hour` | OK |
| `max_prowjob_age` | `168h` (7d) | `7 * 24 * time.Hour` | OK |
| `max_pod_age` | `24h` | `24 * time.Hour` | OK |
| `terminated_pod_ttl` | same as `max_pod_age` | copies `MaxPodAge.Duration` | OK |
| `exclude_clusters` | empty | zero-value slice | OK |

Source: `pkg/config/config.go:2687-2701` (defaults), `:1097-1113` (struct)

### Sinker behaviour claims

| Claim | Status | Source location |
|-------|--------|-----------------|
| Deletes completed non-periodic ProwJobs older than `max_prowjob_age` | OK | `cmd/sinker/main.go:354-375` |
| Keeps latest run for active periodic jobs | OK | `cmd/sinker/main.go:377-411` |
| Deletes pods with label `created-by-prow=true` | OK | `cmd/sinker/main.go:428`, `pkg/kube/prowjob.go:25` |
| Deletes orphaned pods (no matching ProwJob) | OK | `cmd/sinker/main.go:474-478, 535-553` |
| Uses `terminated_pod_ttl` after ProwJob completion | OK | `cmd/sinker/main.go:449-465` |
| Respects `exclude_clusters` | OK | `cmd/sinker/main.go:417-426` |

### CLI flag defaults

| Flag | Documented | Source | Status |
|------|-----------|--------|--------|
| `--run-once` | `false` | `false` | OK |
| `--dry-run` | `true` | `true` | OK |
| `--metrics-port` | `9090` | `9090` | OK |
| `--pprof-port` | `6060` | `6060` | OK |
| `--health-port` | `8081` | `8081` | OK |
| `--profile-memory-usage` | `false` | `false` | OK |
| `--memory-profile-interval` | `30s` | `30s` | OK |

Source: `cmd/sinker/main.go:77-79`, `pkg/flagutil/instrumentation.go:25-29, 53-64`

## Converging Concerns

Issues flagged independently by two or more reviewers.

### Hardcoded defaults may silently drift from source code (2 REVIEWERS)

Flagged by: **Maintainability** and **Deployment Risk**. Both verified the documented defaults are currently accurate, but noted there is no automated mechanism to detect divergence if the code defaults change in the future. This is inherent to hand-written docs and not actionable within this PR.

## Findings

### SUGGESTION: Extract shared flag tables into a reusable partial

The config loading, Kubernetes client, and instrumentation flag tables are common across most Prow components (from `flagutil.KubernetesOptions`, `flagutil.InstrumentationOptions`, `configflagutil.ConfigOptions`). As more placeholder pages get replaced following this template, duplicating these tables creates an N-way update burden.

Consider creating a shared include or reference page before filling in the next component doc. Non-blocking for this PR since sinker is the first real component page.

Flagged by: Maintainability reviewer

### SUGGESTION: Config-loading flags registered but unused by sinker

The flags `--moonraker-address`, `--cache-dir-base`, and `--in-repo-config-cache-size` are inherited from the shared `ConfigOptions` flagset. Sinker calls `ConfigAgent()` which passes `nil` additionals, so these flags are accepted by the binary but have no effect.

Documenting them is not *wrong* (they do appear in `--help`), but may mislead operators. Options:

- Drop these three rows from the table, or
- Add a note like *"inherited from shared config loading; not used by sinker"*

Source: `pkg/flagutil/config/config.go:82-96`

### SUGGESTION: `--config-path` default shown as `""` but it is required

The table says default is `""`, and the description says "(required)". Since validation rejects an empty value, showing `""` as the default is slightly misleading. Consider changing the default column to *none (required)* to avoid ambiguity.

Source: `pkg/flagutil/config/config.go:68-71`

### NIT: `--dry-run` description wording

The doc says "Do not perform mutating Kubernetes API calls" but the source says "Whether or not to make mutating API calls to Kubernetes". The doc's phrasing is clearer for operators, but could be read as "`--dry-run` always prevents mutations" without making it obvious that the default is `true` (mutations suppressed by default).

Source: `cmd/sinker/main.go:79`

## Strengths

- **All factual claims verified correct** — Config defaults, controller behaviour, flag defaults, and label selectors all match the source code exactly.
- **Documents non-obvious behaviour** — The detail about keeping the latest periodic run, the `terminated_pod_ttl` distinction from `max_pod_age`, and the `--dry-run=true` production gotcha are all things an operator would struggle to learn from the code alone.
- **Well-structured flag tables** — Grouping flags by category (sinker, config, kubernetes, instrumentation) makes the page scannable.
- **Source code link aids traceability** — The link to `cmd/sinker` helps future maintainers trace from docs to implementation.
- **Operational safety callout** — The `--dry-run=false` production note and required RBAC permissions reduce the likelihood of misconfiguration incidents for new deployments.

## Verdict

**APPROVE WITH SUGGESTIONS** — All three reviewers approve unanimously. Content is accurate and well-organized. Suggestions are non-blocking polish and a forward-looking concern about shared flag table duplication as more component pages are written.

### Suggested PR comment

This is a clean documentation-only PR that replaces a placeholder with accurate, well-structured sinker component docs. All reviewed defaults and flag values match the current source code. No code, configuration, or deployment changes are involved. I'd suggest considering a shared partial for the common flag tables (config loading, kubernetes, instrumentation) before filling in more component pages, to avoid N-way duplication down the road. Approved — nice contribution to reducing the docs debt.
