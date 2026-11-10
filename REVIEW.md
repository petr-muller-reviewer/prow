---
pr: kubernetes-sigs/prow#945
title: "tide: keep merged PRs returned by the open PR search out of the pool"
head_sha: 7014957aaef5a2bc7cd41a1f99bb68173d6268d3
base: main
reviewed_at: 2026-11-10T01:21:01Z
verdict: approve
refresh_log:
  - at: 2026-11-10T01:21:01Z
    old_sha: 7014957aaef5a2bc7cd41a1f99bb68173d6268d3
    new_sha: 7014957aaef5a2bc7cd41a1f99bb68173d6268d3
    summary: "No code changes; PR merged after two approvals. Recorded Prucek's unanswered question about whether the new metric is needed."
---

## Verdict

Approve. The change filters an authoritative `merged` state at the search-result boundary before it reaches Tide's pool, with bounded observability and a focused regression test. It introduces no configuration, API, or rollout compatibility risk.

## What this PR does

- Requests the pull request `merged` field in Tide's existing GitHub search selection.
- Excludes a search result whose authoritative state is already merged before inserting it into the candidate pool.
- Logs skipped stale results and increments a per-org/repo counter for search-index lag.
- Tests both pool exclusion and the metric increment.

Since previous review:
- No code changes (`7014957a` throughout); the PR merged on 2026-09-22 at 11:44 UTC.
- Reviewed and approved by `Prucek` (2026-09-22 11:19) and `petr-muller` (2026-09-22 11:43); approval bot confirmed and Tide merged it.
- One inline question was raised on `pkg/tide/tide.go:281` and left unanswered before merge; recorded under Open questions.

## Findings

No findings.

## Checked

- `pkg/tide/github.go`: the guard is applied before the result enters the pool, preventing downstream batch construction from seeing a stale merged PR.
- `pkg/tide/tide.go`: the GraphQL field is a scalar in the existing request; the new metric has bounded `org,repo` labels and no extra API calls.
- `pkg/tide/tide_test.go`: `TestQueryIgnoresMergedPRs` covers the observable filter behavior and counter increment.
- Deployment compatibility: no configuration, RBAC, manifests, credentials, or API contract changes.

## Open questions

- `Prucek` asked inline on `pkg/tide/tide.go:281` (2026-09-22 11:18) whether the new `org,repo`-labelled metric is needed at all, suggesting the plain log line may suffice. No reply was posted before the PR merged; the metric shipped as-is. Worth a followup only if the metric turns out to be noise in production dashboards.

## Followups

### deferred-review: apply the merged-PR guard to the status controller's search
- mode: deliver
- where: `pkg/tide/status.go:686-714`, shared helper in `pkg/tide/github.go:171`
- necessity: should
- accepted_at: 2026-11-10T01:24:29Z
- why: `statusController.search()` uses the same `state:open` `ghProvider.search()` results that PR #945 fixed for the pool, but passes them straight to `setStatuses()` without the merged check, so a stale merged PR still gets a computed tide status posted to its head.

```prompt
In kubernetes-sigs/prow, following PR #945 "tide: keep merged PRs returned by the open PR search out of the pool" (merge commit f36bdc5dd8ddbf45118e7b035fe1a2970666bf52, merged to main), apply the same merged-PR guard to the StatusController's search path.

Context: PR #945 added a check in GitHubProvider.Query() (pkg/tide/github.go, around lines 141-148) that skips a search result with pr.Merged == true, logs "Search returned an already merged PR, ignoring it." and increments tideMetrics.searchMerged (tide_search_merged_prs_total{org,repo}, defined in pkg/tide/tide.go around line 274). That check covers only the pool. statusController.search() in pkg/tide/status.go (around lines 652-718) runs the same ghProvider.search() over openPRsQueries (which start with "is:pr state:open ...") and appends every result via CodeReviewCommonFromPullRequest at line ~714, so a PR merged while GitHub's search index lags is still processed by setStatuses() and typically receives a "Not mergeable." tide status on its head.

Task:
1. Make the merged check cover both search consumers. Preferred: move the filter into GitHubProvider.search() (pkg/tide/github.go, around line 171) so every caller inherits it; if you do, keep the status controller's LatestPR watermark logic in mind - it uses result[len(result)-1].UpdatedAt, so dropping trailing merged results will hold the watermark slightly further back (an extra re-fetch of the same window, harmless but worth a comment). Filtering in statusController.search()'s result loop instead is also acceptable if it reads more clearly.
2. Reuse the existing metric and logging: tideMetrics.searchMerged.WithLabelValues(org, repo).Inc() and a warn-level log with the PR's log fields; do not add a second counter or rename the existing one.
3. Add a regression test next to the existing TestStatusControllerSearch (pkg/tide/status_test.go:2333) proving that a merged PR returned by the search does not reach setStatuses() (and, if practical, that the metric increments once). Extend TestQueryIgnoresMergedPRs only if the refactor changes what it asserts.

Acceptance criteria:
- A merged PR returned by statusController.search() is excluded before status computation; a non-merged PR in the same result set is still processed.
- go test ./pkg/tide/ passes, including the new regression test, and golangci-lint is clean.
- No behaviour change for open PRs or for the Query() pool path.

Scope guard:
- Do not change search query strings, pagination, rate-limit accounting, or the LatestPR watermark semantics beyond what the filter requires.
- Do not touch the metric name, labels, or help text; do not refactor unrelated status code.
- Do not reopen or comment on PR #945 or issue #651; this lands as a new change.
```
