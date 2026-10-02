---
pr: kubernetes-sigs/prow#969
title: "tide: add max query concurrency"
head_sha: 7cd8b4253eddb3847465a1586723ef9fb5d9583b
base: main
reviewed_at: 2026-10-02T11:44:29Z
verdict: approve
refresh_log:
  - at: 2026-09-28T15:23:24Z
    from_sha: d40ab8d38539e9a794ab24c55ea27e42fe960361
    to_sha: d40ab8d38539e9a794ab24c55ea27e42fe960361
    summary: "No code changes; incorporated cblecker's five inline comments and review summary."
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
