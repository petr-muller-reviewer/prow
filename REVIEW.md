---
pr: kubernetes-sigs/prow#969
title: "tide: add max query concurrency"
head_sha: 7cd8b4253eddb3847465a1586723ef9fb5d9583b
base: main
reviewed_at: 2026-10-05T20:50:49Z
verdict: approve
refresh_log:
  - at: 2026-09-28T15:23:24Z
    from_sha: d40ab8d38539e9a794ab24c55ea27e42fe960361
    to_sha: d40ab8d38539e9a794ab24c55ea27e42fe960361
    summary: "No code changes; incorporated cblecker's five inline comments and review summary."
  - at: 2026-10-05T20:50:49Z
    from_sha: 7cd8b4253eddb3847465a1586723ef9fb5d9583b
    to_sha: 7cd8b4253eddb3847465a1586723ef9fb5d9583b
    summary: "No code changes; recorded cblecker's approval and the PR merge."
---

# Review

## Gate

The prior gate decision was **Hold** on 2026-09-27 at head `d40ab8d38539e9a794ab24c55ea27e42fe960361`. Its concerns included the unclear limit scope, missing operator documentation, and query-error logging. The PR has since changed; this review covers head `7cd8b4253eddb3847465a1586723ef9fb5d9583b`. The gate itself has not been rerun.

## Verdict

**Approve with suggestions.** The three maintainer perspectives found no blocking issues. The optional setting preserves current behavior by default, and its scope and auth-based sharding are documented. Clarifying that the cap is per Tide process, and that aggregate concurrency grows with replica count, would help operators size it. A low configured cap may lengthen sync and status cycles.

## What this PR does

- Adds `tide.max_query_concurrency` to limit PR-search fan-out independently in the sync and status controllers.
- Keeps zero as unlimited and rejects negative values, preserving existing configuration behavior.
- Documents the per-controller cap, possible combined maximum, and GitHub Apps org sharding.
- Adds tests for configured concurrency, returned PRs, and config parsing.

Since previous review: no code changes. The PR merged on 2026-10-02 at 23:35 UTC.

- `cblecker` submitted an `APPROVED` review on 2026-10-02 at 23:14 UTC.
- `kubernetes-prow[bot]` posted the approval notification at 23:14 UTC.
- No inline review comments were added.

## Findings

### [should-fix] Document replica-scaled concurrency

- where: `site/content/en/docs/components/core/tide/config.md:31-34`
- concern: The guide describes the per-controller cap and the maximum across sync and status, but does not say the limit is local to one Tide process. Multiple replicas multiply the aggregate concurrency, so document that scope to help operators size the setting.
- excerpt: |
    * `max_query_concurrency`: The maximum number of GitHub PR search queries each Tide
       controller (sync and status) runs in parallel. The limit applies per controller,
       so up to twice this number of searches may run at once. Queries are split by org
       only with GitHub Apps authentication. Defaults to 0, which means unlimited.

## Resolved

- **Define the concurrency limit's actual scope:** the config comment and operator guide state that the limit applies separately to sync and status, can allow up to twice the configured number of searches, and shards by org only with GitHub Apps auth.
- **Document the new operator setting:** the Tide configuration guide covers the setting and its unlimited default.
- **Test a positive concurrency limit:** config tests cover negative, zero, and positive values; Tide tests measure limited and unlimited in-flight searches and check returned PRs.
- **Explain why errgroup callbacks return nil:** both search paths explain that the group is used as a limiter while errors are collected separately, and explicitly discard `Wait()` results.
- **Avoid a duplicate Error log for query failures:** the inner sync log is now Debug level and uses provider-neutral wording.

## Checked

- The config parser rejects negative limits; zero remains unlimited.
- The cap is applied per controller. A configured value of `N` can allow up to `2N` searches across sync and status in one Tide process, with additional concurrency from multiple replicas.
- Existing configurations need no migration. If operators add the new field, older strict `checkconfig` tooling may reject it during rollback validation; the normal runtime loader tolerates unknown fields.
- The added config and concurrency tests cover the reviewed paths. Tests were not run during this review.
- No credential, RBAC, or breaking deployment changes were identified.

## Open questions

- For GitHub Apps auth, is a single controller-wide queue across org shards the intended operational tradeoff, given that one org's searches can wait behind another's?

## Followups

### 1. docs — Document replica-scaled concurrency (`should`)

```text
In `kubernetes-sigs/prow`, following merged PR #969 — “tide: add max query concurrency” (merge commit `04b0f796cea5297358c0db2c1a89c349a5d2a2ad`), update the Tide operator guide and the `Tide.MaxQueryConcurrency` comment to make the replica-scaled scope explicit. Explain that sync and status each apply the configured limit within one Tide process, allowing up to 2×N PR search tasks per process, and that aggregate concurrency scales with replica count. Preserve the documented zero-means-unlimited behavior and GitHub Apps org-sharding detail.

Acceptance criteria: the config comment and operator guide agree with the implementation and state the per-process and replica-scaled bounds; no runtime behavior or limit scope changes.

Scope guard: keep this documentation-only; do not change scheduling, defaults, or the concurrency limit implementation.
```

### 2. observability — Measure query limiter wait time (`could`)

```text
In `kubernetes-sigs/prow`, following merged PR #969 — “tide: add max query concurrency” (merge commit `04b0f796cea5297358c0db2c1a89c349a5d2a2ad`), add a Prometheus histogram for the time PR search tasks spend waiting for an `errgroup.Group` concurrency slot in `GitHubProvider.Query` (`pkg/tide/github.go`) and `statusController.search` (`pkg/tide/status.go`). The existing per-search duration starts after a task begins, so it omits time blocked while `g.Go` waits for a slot. Use bounded labels that distinguish the controller without labeling raw query strings, register the metric with Tide's existing metrics, and add deterministic coverage for a saturated limit and the unlimited case.

Acceptance criteria: the metric includes submission-to-worker-start delay for both search paths, has bounded label cardinality, and tests demonstrate observations when a limit of 1 queues work while leaving scheduling results unchanged.

Scope guard: do not change concurrency defaults, scheduling/fairness semantics, or unrelated Tide metrics.
```
