---
pr: kubernetes-sigs/prow#968
title: "tide: add query observability metrics"
head_sha: f34ab19e7bb9ba3036793a6f6b45d14b0cba0bc3
base: main
reviewed_at: 2026-10-02T11:50:55Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-10-02T11:48:55Z
  gated_head_sha: f34ab19e7bb9ba3036793a6f6b45d14b0cba0bc3
  reviewed_head_sha: f34ab19e7bb9ba3036793a6f6b45d14b0cba0bc3
---

# Review

## Gate

**Decision: do-not-merge.** A full re-review against `upstream/main` at `6844e439e237641e025f0e094833768c6928bc00` found that the PR head is unchanged and all blocking findings remain. PR #982 changed Tide's GraphQL timeout and retry path on `main`; the combined tree merges cleanly and its Tide and GitHub package tests pass, but retries do not address the metric contract or stale gauges. There are no substantive GitHub reviews or comments resolving these concerns.

Gating findings:

- **Not addressed — existing error series** (`REVIEW.md`, `pkg/tide/github.go:127-129`): a paginated search that returns PRs before failing still records `tidequeryresults{result="partial"}` instead of `result="error"`. Preserve the old series before merge.
- **Not addressed — empty-cycle completeness** (`REVIEW.md`, `pkg/tide/github.go:172-178`, `pkg/tide/status.go:639-644`): the ratio remains stale when no shards run, and status returns before resetting gauges when all queries are removed. Define and publish the empty-cycle state before merge.
- **Not addressed — tests and metric reference** (`REVIEW.md`, `pkg/tide/github.go:183-190`, `pkg/tide/tide.go:308-349`): this PR adds no focused tests for the new outcomes and does not add the six series to `site/content/en/docs/metrics/_index.md`. Add both before merge.

Independent merge risk: PR #968 adds no exported API, Tide configuration, or permission changes. Its existing `tidequeryresults` label semantics do change for terminal partial failures, including failures after PR #982's retries are exhausted; deployments with error-only alert queries can silently undercount them. Recovered gateway timeouts appear in duration but not the classified error counter, because that counter observes the final logical search result.

## Verdict

Request changes before merging.

The Code Quality, Maintainability, and Deployment Risk reviews agree that this change can hide terminal partial failures from existing error monitoring and leave the new completeness gauge stale after queries are removed. The monitoring risk is high because existing error-only alerts can silently undercount failures. The new metric contract also needs focused tests and an update to the metrics reference; PR #982's retry behavior on current `main` introduces no additional confirmed blocker.

## What this PR does

- Measures the duration and returned PR count of each GitHub search shard in the sync and status controllers.
- Counts query errors by a bounded error class and counts searches that return some PRs before failing.
- Publishes per-cycle shard outcome counts and a success-shard ratio for each controller.
- Extends the existing sync query result counter with a `partial` outcome.

## Findings

### [blocking] Preserve the existing query error series

- where: `pkg/tide/github.go:127-129`
- concern: A search that returns one page of PRs and then fails, including after PR #982's gateway timeout retries are exhausted, now increments `tidequeryresults{result="partial"}` instead of its existing `result="error"` series. Existing error-rate alerts and dashboards will miss these terminal failures. Keep the old counter's success/error meaning and use the new partial-results counter for the extra distinction.
- excerpt: |
    result := queryResult(err, len(results))
    queryID := strconv.Itoa(i)
    tideMetrics.queryResults.WithLabelValues(queryID, org, result).Inc()

### [blocking] Reset completeness when no shards run

- where: `pkg/tide/github.go:172-178`
- concern: If the sync controller has no query shards, the shard gauges become zero while `poolCompletenessRatio` retains its previous value. The status path also returns before updating any of these gauges when no Tide queries are configured (`pkg/tide/status.go:639-644`). After a configuration change, dashboards can show completeness for a cycle that did not run; define and publish a value for this case in both paths.
- excerpt: |
    total := shardSuccess + shardPartial + shardError
    tideMetrics.syncQueryShards.WithLabelValues(controller, "success").Set(float64(shardSuccess))
    tideMetrics.syncQueryShards.WithLabelValues(controller, "partial").Set(float64(shardPartial))
    tideMetrics.syncQueryShards.WithLabelValues(controller, "error").Set(float64(shardError))
    if total > 0 {
        tideMetrics.poolCompletenessRatio.WithLabelValues(controller).Set(float64(shardSuccess) / float64(total))
    }

### [blocking] Test the new query outcomes and metric values

- where: `pkg/tide/github.go:183-190`
- concern: Current `main` tests search pagination and gateway retries, but neither those tests nor this PR assert the metric values for partial-page failures or cycles with no shards. Add focused behavior tests that assert the old error counter, the new partial and classified error counters, and the empty-cycle gauges in both controllers.
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

### [blocking] Document the added metrics

- where: `pkg/tide/tide.go:308-349`
- concern: The metrics reference at `site/content/en/docs/metrics/_index.md:22` omits all six new series. Add their types, labels, and meanings so operators know what to query; retain the existing `tidequeryresults` success/error contract in both code and documentation.
- excerpt: |
    queryDuration: prometheus.NewHistogramVec(prometheus.HistogramOpts{
        Name: "tide_query_duration_seconds",
        Help: "Duration of individual Tide GitHub search queries per shard.",
    }, []string{
        "controller",
        "result",
    }),

### [nit] Consolidate repeated query outcome accounting

- where: `pkg/tide/status.go:724-746`
- concern: The status and sync controllers repeat the same outcome switch and final gauge publication (`pkg/tide/github.go:141-148,172-178`). A small shared recorder would keep their outcome and label behavior aligned as these metrics evolve. This is a maintainability judgment call; the repository has no documented rule requiring it.
- excerpt: |
    switch resultLabel {
    case "error":
        shardError++
    case "partial":
        shardPartial++
    default:
        shardSuccess++
    }

### [nit] Clarify the shard gauge name or Help text

- where: `pkg/tide/tide.go:340-345`
- concern: `tide_sync_query_shards` also records `controller="status"`, so its name and Help text suggest a narrower scope than the data it contains. Clarify the contract for operators.
- excerpt: |
    syncQueryShards: prometheus.NewGaugeVec(prometheus.GaugeOpts{
        Name: "tide_sync_query_shards",
        Help: "Number of query shards in the most recent sync cycle by outcome.",
    }, []string{
        "controller",
        "result",
    }),

### [nit] Narrow error class matching where possible

- where: `pkg/tide/github.go:219-221`
- concern: The fallback classification matches broad message fragments, including bare HTTP status numbers, so unrelated text can be classified as a server error. Prefer structured error details where available, or match the known client error format more narrowly; keep a documented fallback for unstructured errors.
- excerpt: |
    if strings.Contains(msg, "502") || strings.Contains(msg, "503") || strings.Contains(msg, "504") || strings.Contains(msg, "500") {
        return "server_error"
    }

## Resolved

None.

## Checked

- Inspected both Tide search paths and the shared paginated search helper.
- Re-reviewed the full PR diff against `upstream/main` at `6844e439e237641e025f0e094833768c6928bc00`, including PR #982's gateway timeout and page-size retry changes. A synthetic merge was clean; no new integration defect was found.
- Independently checked Code Quality, Maintainability, and Deployment Risk findings against the PR head; all three reviews agree on the error-series and empty-cycle problems.
- Confirmed the new histogram and counter label sets are bounded by controller, configured query index or org shard, result, and fixed error classes.
- Checked that the PR adds no Tide configuration fields, permissions, external services, or GitHub API calls.
- `go test ./pkg/tide -run 'TestQueryShardsByOrgWhenAppsAuthIsEnabledOnly|TestQueryIgnoresMergedPRs|TestSearch' -count=1` passed.
- `git diff --check` passed for PR #968's effective diff against current `main`.
- `go test ./pkg/tide ./pkg/github` passed on the synthetic merge of PR #968 with current `main`.

## Open questions

- What value should `tide_pool_completeness_ratio` report when no query shards are configured: zero, or no series? Please make the choice explicit in code and the metric documentation.
- Should the classified error counter represent only the final logical search outcome after PR #982's retries, as it does now, or also count recovered gateway timeouts? Please document the intended meaning so operators can distinguish final failures from retry pressure.
