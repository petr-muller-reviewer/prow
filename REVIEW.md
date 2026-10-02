---
pr: kubernetes-sigs/prow#969
title: "tide: add max query concurrency"
head_sha: 7cd8b4253eddb3847465a1586723ef9fb5d9583b
base: main
reviewed_at: 2026-10-02T09:16:05Z
verdict: approve
refresh_log:
  - at: 2026-09-28T15:23:24Z
    from_sha: d40ab8d38539e9a794ab24c55ea27e42fe960361
    to_sha: d40ab8d38539e9a794ab24c55ea27e42fe960361
    summary: "No code changes; incorporated cblecker's five inline comments and review summary."
---

# Review

## Gate

The prior gate decision was **Hold** on 2026-09-27 at head `d40ab8d38539e9a794ab24c55ea27e42fe960361`. Its concerns included the unclear limit scope, missing operator documentation, and the query-error log. The PR has since changed; this review covers head `7cd8b4253eddb3847465a1586723ef9fb5d9583b`. The gate itself has not been rerun.

## Verdict

**Approve with suggestions.** The three maintainer perspectives found no blocking issues, and deployment risk is low. The setting preserves current behavior by default, its per-controller scope and org sharding are documented, and tests cover limited and unlimited concurrency. As a non-blocking observability improvement, include the org in status-search failure context; operators who set a low cap should watch sync and status refresh duration.

## What this PR does

- Adds `tide.max_query_concurrency` to limit PR-search fan-out independently in the sync and status controllers.
- Keeps zero as unlimited and rejects negative values, preserving existing configuration behavior.
- Documents the per-controller cap, possible combined maximum, and GitHub Apps org sharding.
- Adds tests for configured concurrency, returned PRs, and config parsing.

## Findings

### [nit] Include org context in status-search failures

- where: `pkg/tide/status.go:681-682`
- concern: Status searches run once per org when GitHub Apps auth is enabled, but the per-query log context records only the query and errors are later aggregated. Adding the org as a structured field or wrapping each error with its org would make failures easier to identify.
- excerpt: |
    for org, query := range queries {
        g.Go(func() error {
            now := time.Now()
            log := sc.logger.WithField("query", query)

## Resolved

- **Define the concurrency limit's actual scope:** the config comment and operator guide now say the limit applies separately to sync and status, can allow up to twice the configured number of searches, and shards by org only with GitHub Apps auth.
- **Document the new operator setting:** the Tide configuration guide now covers the setting and its unlimited default.
- **Test a positive concurrency limit:** config tests cover negative, zero, and positive values; Tide tests measure limited and unlimited in-flight searches and check returned PRs.
- **Explain why errgroup callbacks return nil:** both search paths explain that the group is used as a limiter while errors are collected separately, and explicitly discard `Wait()` results.
- **Avoid a duplicate Error log for query failures:** the inner sync log is now Debug level and uses provider-neutral wording.

## Checked

- The config parser rejects negative limits; zero remains unlimited.
- The cap is applied per controller. A configured value of `N` can allow up to `2N` searches across sync and status in one Tide instance.
- Existing configurations require no migration. If operators add the new field, older strict-config binaries may reject it during validation.
- The added config and concurrency tests cover the reviewed paths. Tests were not run during this review.
- No breaking deployment changes, credential changes, or new permissions were identified.

## Open questions

- For GitHub Apps auth, is a single controller-wide queue across org shards the intended operational tradeoff, given that one org's searches can wait behind another's?
