---
pr: kubernetes-sigs/prow#968
title: "tide: add query observability metrics"
head_sha: e7e19680f0f09b538a44a30b7399c622da674de5
base: main
reviewed_at: 2026-10-05T10:12:55Z
verdict: request-changes
refresh_log:
  - at: 2026-10-02T12:07:46Z
    old_sha: f34ab19e7bb9ba3036793a6f6b45d14b0cba0bc3
    new_sha: f34ab19e7bb9ba3036793a6f6b45d14b0cba0bc3
    summary: "No code changes; incorporated four inline maintainer comments and a COMMENTED review."
  - at: 2026-10-05T10:12:55Z
    old_sha: f34ab19e7bb9ba3036793a6f6b45d14b0cba0bc3
    new_sha: e7e19680f0f09b538a44a30b7399c622da674de5
    summary: "Preserved legacy query error labels, shared shard accounting, clarified the shard gauge, and documented the metrics; four findings resolved."
gate:
  decision: do-not-merge
  gated_at: 2026-10-05T10:15:09Z
  gated_head_sha: e7e19680f0f09b538a44a30b7399c622da674de5
  reviewed_head_sha: e7e19680f0f09b538a44a30b7399c622da674de5
---

# Review

## Gate

**Decision: do-not-merge.** PR #968 is open at `e7e19680f0f09b538a44a30b7399c622da674de5`, the same head covered by the refreshed review. Later code addressed the legacy error-series concern, added metric documentation, shared shard accounting, and renamed the shard gauge. Two blocking findings remain: completeness can retain stale values after the query set becomes empty, and the PR adds no focused assertions for its new metric behavior. The GitHub reviews are `COMMENTED`; the substantive comments about the metric docs, shared accounting, gauge name, and partial-error semantics are addressed by the current patch.

Gating findings:

- **Not addressed — empty-cycle completeness** (`REVIEW.md`, `pkg/tide/github.go:199-205`, `pkg/tide/status.go:643-644,740-741`): `queryShardCounts.report` sets the ratio only when `total > 0`, and status returns before reporting any gauges when no queries are configured. After removing all queries, the completeness ratio can remain from an earlier cycle. Define the empty-cycle value or clear the series in both paths before merge.
- **Not addressed — metric behavior tests** (`REVIEW.md`; current PR files include no test changes): the metrics for partial failures, classified errors, legacy counter semantics, and empty cycles have no focused assertions in either controller. Add targeted tests for those outcomes before merge.

Other review finding: **Narrow error class matching where possible** (`REVIEW.md`, `pkg/tide/github.go:228-247`) remains a nit. Matching bare `500`–`504` substrings can misclassify unrelated error text as `server_error`; it does not independently block merge.

Independent merge risk: the patch changes only Tide metric collection and the metrics reference. It adds no exported API, configuration field, command-line flag, Kubernetes schema, or wire format, and it preserves the existing `tidequeryresults` success/error labels. Existing Tide deployments gain new Prometheus series; no existing series is removed or redefined. No notable compatibility risk was found.

## Verdict

Request changes before merging. This refresh resolves the legacy error-series, metrics documentation, shared-accounting, and gauge naming findings. The empty-cycle completeness behavior and missing focused metric tests remain blocking; the broad error-class string matching remains a nit.

The earlier Code Quality, Maintainability, and Deployment Risk reviews agreed that the original patch could hide terminal partial failures from existing error monitoring and leave the completeness gauge stale after queries were removed. The refreshed patch fixes the legacy error-series behavior and adds the metrics reference. The empty-cycle case and focused metric tests remain unresolved; PR #982's retry behavior on current `main` introduces no additional confirmed blocker.

## What this PR does

- Measures the duration and returned PR count of each GitHub search shard in the sync and status controllers.
- Counts query errors by a bounded error class and counts searches that return some PRs before failing.
- Publishes per-cycle shard outcome counts and a success-shard ratio for each controller.
- Preserves the existing query result counter's success/error outcomes while exposing partial results through separate metrics.

Since the previous refresh on 2026-10-02:

- No code changed. At 12:02–12:07 UTC on 2026-10-02, @petr-muller left four inline comments about the metrics reference, duplicated shard accounting, the `sync` gauge name, and how partial failures were previously counted; the submitted review was `COMMENTED`.
- These comments align with existing findings. The pre-PR `tidequeryresults` counter recorded every non-nil search error, including a partial result, as `result="error"`.

Since previous review:

- The PR head moved from `f34ab19e7bb9ba3036793a6f6b45d14b0cba0bc3` to `e7e19680f0f09b538a44a30b7399c622da674de5`. Its focused patch now preserves `tidequeryresults` success/error labels for partial failures, shares shard accounting between controllers, renames the shard gauge to `tide_query_shards`, and documents the six new metrics.
- On 2026-10-02 at 13:54 UTC, `kubernetes-prow[bot]` posted an approval-status notice. At 13:57, @Prucek asked whether the metrics reference could be generated and submitted a `COMMENTED` review; at 14:00, @Prucek confirmed partial failures were previously counted as errors, submitted another `COMMENTED` review, and thanked the author for the update. At 14:36, @petr-muller said the reference could probably be generated and asked whether to pursue it, submitting a third `COMMENTED` review.

## Findings

### [blocking] Reset completeness when no shards run

- where: `pkg/tide/github.go:199-205`, `pkg/tide/status.go:643-644,740-741`
- concern: If the sync controller has no query shards, the shard gauges become zero while `poolCompletenessRatio` retains its previous value because `report` only sets the ratio when `total > 0`. The status path returns before updating any of these gauges when no Tide queries are configured. The new documentation describes the ratio as applying to a cycle with at least one shard, but dashboards can still show the previous ratio after all queries are removed; define and publish an explicit empty-cycle value or clear the series in both paths.
- excerpt: |
    func (s *queryShardCounts) report(controller string) {
        tideMetrics.queryShards.WithLabelValues(controller, "success").Set(float64(s.success))
        tideMetrics.queryShards.WithLabelValues(controller, "partial").Set(float64(s.partial))
        tideMetrics.queryShards.WithLabelValues(controller, "error").Set(float64(s.failed))
        if total := s.success + s.partial + s.failed; total > 0 {
            tideMetrics.poolCompletenessRatio.WithLabelValues(controller).Set(float64(s.success) / float64(total))
        }
    }

### [blocking] Test the new query outcomes and metric values

- where: `pkg/tide/github.go:208-215`, `pkg/tide/status.go:640-644`; no test files are included in the updated PR patch.
- concern: The updated PR still adds no focused assertions for its metric values. Existing search tests do not verify partial-page failures against the legacy error counter, the new partial and classified error counters, or empty-cycle gauges in both controllers.
- excerpt: |
    func queryResult(err error, resultCount int) string {
        if err == nil {
            return "success"
        }
        if resultCount == 0 {
            return "error"
        }
        return "partial"
    }

### [nit] Narrow error class matching where possible

- where: `pkg/tide/github.go:228-247`
- concern: The fallback classification matches broad message fragments, including bare HTTP status numbers, so unrelated text can be classified as a server error. Prefer structured error details where available, or match the known client error format more narrowly; keep a documented fallback for unstructured errors.
- excerpt: |
    if strings.Contains(msg, "502") || strings.Contains(msg, "503") || strings.Contains(msg, "504") || strings.Contains(msg, "500") {
        return "server_error"
    }

## Resolved

### [blocking] Preserve the existing query error series

- where: `pkg/tide/github.go:127-129` at the previous review
- concern: A search that returned PRs before failing was changed to increment `tidequeryresults{result="partial"}` instead of its existing `result="error"` series, risking undercounts in existing error alerts.
- resolution: The updated PR derives the legacy counter value from `err`, preserving its success/error semantics while recording partial outcomes through the new metrics.
- excerpt: |
    resultString := "success"
    if err != nil {
        resultString = "error"
    }
    tideMetrics.queryResults.WithLabelValues(queryID, org, resultString).Inc()

### [blocking] Document the added metrics

- where: `pkg/tide/tide.go:308-349` at the previous review
- concern: The metrics reference omitted the six added series and their labels and meanings.
- resolution: `site/content/en/docs/metrics/_index.md:24-29` now documents all six metrics and their labels and meanings.

### [nit] Consolidate repeated query outcome accounting

- where: `pkg/tide/status.go:724-746` and `pkg/tide/github.go:141-148,172-178` at the previous review
- concern: The status and sync controllers repeated the same outcome switch and final gauge publication.
- resolution: Both paths now use `queryShardCounts.add` and `queryShardCounts.report` from `pkg/tide/github.go:183-205`.

### [nit] Clarify the shard gauge name or Help text

- where: `pkg/tide/tide.go:340-345` at the previous review
- concern: `tide_sync_query_shards` also recorded the status controller, so its name implied a narrower scope than its data.
- resolution: The series is now named `tide_query_shards` and its Help text refers to the most recent search cycle.

## Checked

- Considered overlap between the proposed metrics and the existing `tidequeryresults` counter. The maintainer accepted retaining the broader metric set, so metric count alone is not a review finding.
- Inspected both Tide search paths and the shared paginated search helper.
- Re-reviewed the full PR diff against `upstream/main` at `6844e439e237641e025f0e094833768c6928bc00`, including PR #982's gateway timeout and page-size retry changes. A synthetic merge was clean; no new integration defect was found.
- Independently checked Code Quality, Maintainability, and Deployment Risk findings against the PR head; all three reviews agree on the error-series and empty-cycle problems.
- Confirmed the new histogram and counter label sets are bounded by controller, configured query index or org shard, result, and fixed error classes.
- Checked that the PR adds no Tide configuration fields, permissions, external services, or GitHub API calls.
- `go test ./pkg/tide -run 'TestQueryShardsByOrgWhenAppsAuthIsEnabledOnly|TestQueryIgnoresMergedPRs|TestSearch' -count=1` passed.
- `git diff --check` passed for PR #968's effective diff against current `main`.
- `go test ./pkg/tide ./pkg/github` passed on the synthetic merge of PR #968 with current `main`.

## Open questions

- When no shards are configured, should `tide_pool_completeness_ratio` retain the previous non-empty cycle's value (as the new documentation describes), or should the code reset or clear it in both controllers?
- Should the classified error counter represent only the final logical search outcome after PR #982's retries, as it does now, or also count recovered gateway timeouts? Please document the intended meaning so operators can distinguish final failures from retry pressure.

## Followups

Accepted 3 followups; skipped 1. PR #968 merged on 2026-10-05 at merge commit `27fe62bda55fff7c69d80f526129ff5e7cc556a5`.

### Empty-cycle completeness metrics — bugfix, must

```text
In kubernetes-sigs/prow, as a post-merge followup to PR #968 — "tide: add query observability metrics" (merge commit 27fe62bda55fff7c69d80f526129ff5e7cc556a5), fix stale Tide query metrics when no shards run. In `pkg/tide/github.go`, `queryShardCounts.report` leaves `tide_pool_completeness_ratio` unchanged when the shard total is zero. In `pkg/tide/status.go`, `statusController.search` returns before reporting any query gauges when no Tide queries are configured.

Choose and implement explicit empty-cycle semantics consistently for the sync and status controllers. Ensure zero-query and zero-shard cycles cannot leave prior query-shard or completeness values visible; document the chosen meaning and add regression tests for both paths.

Acceptance criteria: tests show the selected metric state after a populated cycle is followed by an empty cycle in both controllers; the metrics reference describes the empty-cycle behavior; focused Tide tests pass.

Scope guard: do not change query scheduling, error classification, or metric names; keep the change limited to empty-cycle gauge behavior, its documentation, and regression tests.
```

### Query outcome metric tests — tests, must

```text
In kubernetes-sigs/prow, as a post-merge followup to PR #968 — "tide: add query observability metrics" (merge commit 27fe62bda55fff7c69d80f526129ff5e7cc556a5), add focused tests for the new query outcome metrics in `pkg/tide/github_test.go` and `pkg/tide/status_test.go`.

Cover successful, terminal-error, and partial-result searches in the sync and status paths. Assert the new duration and returned-PR observations, classified-error and partial-result counters, and the legacy `tidequeryresults` success/error contract where that counter is emitted. Do not duplicate empty-cycle scenarios; those are covered by the separate empty-cycle followup.

Acceptance criteria: the tests fail if partial failures stop counting as legacy errors, if the new classified or partial counters use the wrong labels/counts, or if either controller records the wrong duration/result or PR-count observations; focused Tide tests pass.

Scope guard: add test coverage only; do not change metric behavior, add retry-attempt instrumentation, or cover empty-cycle semantics here.
```

### Document final query error and retry semantics — docs, could

```text
In kubernetes-sigs/prow, as a post-merge followup to PR #968 — "tide: add query observability metrics" (merge commit 27fe62bda55fff7c69d80f526129ff5e7cc556a5), clarify the retry semantics of `tide_query_errors_total` in `site/content/en/docs/metrics/_index.md`.

The counter increments only when the completed search returns an error (`pkg/tide/github.go` and `pkg/tide/status.go`); retries recovered inside the search/client path are not counted. State that the counter reflects final logical search errors and does not measure recovered retry attempts. Keep the duration metric's coverage of the full search call clear if useful.

Acceptance criteria: an operator can distinguish final shard failures from transient retries that recovered, and the documentation matches the current instrumentation.

Scope guard: documentation only; do not add retry-attempt metrics or change the search/retry behavior.
```
