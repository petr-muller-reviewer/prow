---
pr: kubernetes-sigs/prow#945
title: "tide: keep merged PRs returned by the open PR search out of the pool"
head_sha: 7014957aaef5a2bc7cd41a1f99bb68173d6268d3
base: main
reviewed_at: 2026-09-22T00:26:21Z
verdict: approve
---

## Verdict

Approve. The change filters an authoritative `merged` state at the search-result boundary before it reaches Tide's pool, with bounded observability and a focused regression test. It introduces no configuration, API, or rollout compatibility risk.

## What this PR does

- Requests the pull request `merged` field in Tide's existing GitHub search selection.
- Excludes a search result whose authoritative state is already merged before inserting it into the candidate pool.
- Logs skipped stale results and increments a per-org/repo counter for search-index lag.
- Tests both pool exclusion and the metric increment.

## Findings

No findings.

## Checked

- `pkg/tide/github.go`: the guard is applied before the result enters the pool, preventing downstream batch construction from seeing a stale merged PR.
- `pkg/tide/tide.go`: the GraphQL field is a scalar in the existing request; the new metric has bounded `org,repo` labels and no extra API calls.
- `pkg/tide/tide_test.go`: `TestQueryIgnoresMergedPRs` covers the observable filter behavior and counter increment.
- Deployment compatibility: no configuration, RBAC, manifests, credentials, or API contract changes.

## Open questions

None.
