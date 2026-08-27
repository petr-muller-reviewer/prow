---
pr: kubernetes-sigs/prow#821
title: "feat(tide): shard large queries to avoid GitHub resource limits"
head_sha: 2151f08e421277505a62f50bbe7175f9b4d890db
base: main
reviewed_at: 2026-08-08T11:22:48Z
verdict: request-changes
---

## Summary

Shards Tide status-controller GitHub queries into per-org(-shard) pieces to avoid GitHub GraphQL resource-limit failures on large orgs, adds `MaxQueryConcurrency` to bound query fan-out in both status and sync controllers, and adds new per-query/per-shard observability metrics. HEAD commit (`2151f08e4`) is explicitly titled `TEMP: default max_query_concurrency to 25` and its own message says it should be dropped before merging upstream.

## Findings

### [blocking] non-GitHub-Apps auth path re-merges sharded queries into one
- where: `pkg/tide/status.go:650-662`
- concern: When `!sc.usesGitHubAppsAuth`, all shard entries from `openPRsQueries` are concatenated back into a single combined query under key `""`, completely undoing the sharding this PR implements. Any non-GitHub-Apps install with a large org still issues one oversized query and hits the exact resource-limit failures this PR is meant to fix.
- excerpt: |
    if !sc.usesGitHubAppsAuth {
    	var orgs []string
    	for org := range queries {
    		orgs = append(orgs, org)
    	}
    	sort.Strings(orgs)
    	var query strings.Builder
    	for _, org := range orgs {
    		query.WriteString(" " + queries[org])
    	}
    	queries = map[string]string{"": query.String()}
    }

### [blocking] query-changed reset targets the wrong storedState key
- where: `pkg/tide/status.go:689-695`
- concern: `latestPR` is read from `sc.storedState[org]` (bare org key), but the "query changed" detection/reset writes to `sc.storedState[shardKey]` (`"org#N"`). For a sharded org, the reset never zeroes the value actually passed into `sc.ghProvider.search`, and `sc.storedState[org].PreviousQuery` is never populated — so the intended full resync after a query change (e.g. an org exception added/removed) never happens for sharded orgs. This is exactly the >100-repo scenario the PR targets.
- excerpt: |
    sc.storedStateLock.Lock()
    latestPR := sc.storedState[org].LatestPR
    if query != sc.storedState[shardKey].PreviousQuery {
    	log.WithField("previously", sc.storedState[shardKey].PreviousQuery).Info("Query changed, resetting start time to zero")
    	sc.storedState[shardKey] = storedState{PreviousQuery: query}
    }
    sc.storedStateLock.Unlock()

### [blocking] quiet/failing shard doesn't gate its org's shared watermark
- where: `pkg/tide/status.go:679-773`
- concern: `shardLatest` is only populated for shards with `len(result) > 0`; the post-`Wait` aggregation takes the minimum only over reporting shards. A shard that is quiet or errors doesn't hold back the org's `LatestPR`, which sibling shards can still advance. On that shard's next run it reads the sibling-advanced, org-shared cursor as its own `since` bound, risking skipped PRs if GitHub's search index has any propagation lag for that shard's repos — the `-30s` margin only protects a shard against its own lag, not against being pulled forward by a sibling.
- excerpt: |
    if len(result) > 0 {
    	latest := result[len(result)-1].UpdatedAt
    	if !latest.IsZero() {
    		shardLatest[shardKey] = latest.Add(-30 * time.Second)
    	}
    }
    ...
    for shardKey, t := range shardLatest {
    	org := shardOrg(shardKey)
    	if prev, ok := orgMinLatest[org]; !ok || t.Before(prev) {
    		orgMinLatest[org] = t
    	}
    }

### [blocking] TEMP commit makes MaxQueryConcurrency=0 (unlimited) unreachable
- where: `pkg/config/config.go:2717-2721`
- concern: `parseProwConfig` unconditionally coerces `MaxQueryConcurrency == 0` to `25` before the `if limit > 0 { g.SetLimit(limit) }` checks ever see it, so the documented "0 means unlimited" behavior is unreachable. This is introduced by commit `2151f08e4`, titled `TEMP: default max_query_concurrency to 25` — its own message says it should be dropped before merging. Every existing install that doesn't set `max_query_concurrency` silently drops from unlimited fan-out to a hard cap of 25 with no way to restore old behavior.
- excerpt: |
    if c.Tide.MaxQueryConcurrency == 0 {
    	c.Tide.MaxQueryConcurrency = 25
    }
    if c.Tide.MaxQueryConcurrency < 0 {
    	return fmt.Errorf("tide has invalid max_query_concurrency (%d), it must be non-negative", c.Tide.MaxQueryConcurrency)
    }

### [blocking] PR description does not match shipped implementation
- where: PR description vs. `pkg/tide/status.go`
- concern: Description claims new `ShardQuery`/`ShardQueries` functions were added to `config.TideQuery`, partitioning by `orgs`/`repos` fields. The actual code is an unexported `shardQueries(map[string]string, int)` helper in `pkg/tide/status.go` that partitions already-rendered query *strings*, not structured config, and is wired only into the status controller (not the sync controller, despite the description's claim). Combined with the non-GitHub-Apps re-merge and sync-path gap above, the description overstates what actually ships.
- excerpt: |
    // description: "Add ShardQuery and ShardQueries functions to config.TideQuery
    //  that partition queries based on orgs and repos fields"
    // actual: func shardQueries(queries map[string]string, maxRepos int) map[string]string { ... }
    //   in pkg/tide/status.go — unexported, string-based, not on config.TideQuery.

### [should-fix] org_shard metric label never carries shard granularity
- where: `pkg/tide/tide.go:276,313-329`, `pkg/tide/github.go:127-151`, `pkg/tide/status.go:713-717`
- concern: `queryErrors`/`queryPartialResults` declare an `org_shard` label, but both callers pass the bare org (`shardOrg(shardKey)` in status.go, plain `org` in github.go) rather than the actual shard key. Despite sharding being this PR's headline feature, the new metrics can't distinguish which shard of a multi-shard org is failing.
- excerpt: |
    tideMetrics.queryErrors.WithLabelValues(controller, "", org, classifyQueryError(err)).Inc()

### [should-fix] shardQueries recovers structure by re-parsing rendered query strings
- where: `pkg/tide/status.go:827-861` (`shardQueries`)
- concern: Sharding is implemented via `strings.Fields` plus a `repo:"` prefix check on the already-formatted GitHub search query string, re-deriving structure that was known before rendering (in `orgRepoQueryStrings`/`constructQuery`). Any future change to token formatting (quoting style, a token legitimately starting with `repo:"`, negated `-repo:` filters interacting differently) would silently mis-shard with no compiler/type-level signal.
- excerpt: |
    for _, tok := range strings.Fields(query) {
    	if strings.HasPrefix(tok, `repo:"`) {
    		repoTokens = append(repoTokens, tok)
    	} else {
    		prefix = append(prefix, tok)
    	}
    }

### [should-fix] sync-controller path lacks repo-count sharding
- where: `pkg/tide/github.go` `GitHubProvider.Query()` (~lines 115-124), `pkg/config/tide.go:662` (`OrgQueries`)
- concern: Only the status controller's `openPRsQueries` is wired through `shardQueries`/`maxReposPerQuery`. The sync controller still uses `query.OrgQueries()`, sharding only by org, not repo count — a single org with >100 repos in one Tide query can still hit GitHub's resource limits on the merge-decision path.

### [nit] classifyQueryError uses unanchored substring match on status codes
- where: `pkg/tide/github.go:257-285`
- concern: `strings.Contains(msg, "500"/"502"/"503"/"504")` against the full error text can misclassify unrelated errors (e.g. "query cost 502 exceeds budget", or a URL containing port 8500) as `server_error`, skewing dashboards/alerts keyed on `error_class`.
- excerpt: |
    if strings.Contains(msg, "500") || strings.Contains(msg, "502") || ...

### [nit] dead PreviousQuery field at org-level storedState key
- where: `pkg/tide/status.go:767-770`
- concern: With reset logic moved to per-shard keys, `sc.storedState[org].PreviousQuery` is only ever copied to itself — a no-op. Either remove it from org-level usage or restructure so the same struct type isn't reused with different semantics at two key granularities.

### [nit] maxReposPerQuery is a bare hardcoded constant
- where: `pkg/tide/status.go` (`maxReposPerQuery = 100`)
- concern: No cross-reference to GitHub's documented query/resource limits, and no config knob, despite this PR adding a related config field (`MaxQueryConcurrency`).

### [question] is the TEMP commit meant to be dropped or squashed?
- concern: `2151f08e4` silently changes behavior for every existing install and contradicts the "0 = unlimited" doc — is 25 meant to become the permanent default, or should this commit be dropped entirely before merge as its own message says?

### [question] was the non-GitHub-Apps re-merge intentional?
- where: `pkg/tide/status.go:650-662`
- concern: Was collapsing shards back into one query for non-GitHub-Apps auth a deliberate, documented limitation of that auth mode, or a leftover from before sharding was introduced that should also shard per-org?

### [question] was the org-level watermark/reset logic designed for multi-shard orgs at all?
- concern: The reset-key mismatch and cross-shard watermark gating gap both suggest the `storedState` design predates per-shard state and wasn't fully updated. Is there an intended design for per-shard state tracking that differs from what's shipped?

### [question] should sync-controller sharding land in this PR or a follow-up?
- concern: The PR title/description imply both controllers are covered; only status is. Should scope be narrowed explicitly, or is sync-path sharding planned as a fast-follow?

## Checked
- Sharding logic (`shardQueries`, `openPRsQueries`) itself looks correct for the GitHub-Apps-auth path: non-repo qualifiers (including `-repo:"..."` exclusions) are preserved verbatim per shard.
- Concurrency bound (`g.SetLimit(limit)`) is wired correctly through errgroup in both sync and status controllers, with a `limit > 0` guard avoiding a `SetLimit(0)` deadlock.
- Per-shard duration/PR-count histograms (`queryDuration`, `queryPRsReturned`) are recorded correctly.
- `MaxQueryConcurrency` config field is optional, `omitempty`, and validated to reject negative values — existing configs otherwise parse unchanged.
- New Prometheus metrics are additive; no existing metric is renamed or removed.
- Persisted `storedState` remains `map[string]storedState`; old state files still deserialize under the new code (per-shard keys like `org#0` accumulate write-only and are never cleaned up if an org later shrinks below the shard threshold — low severity, not a data-loss risk).
- Test coverage for sharding math and status-controller integration (`TestShardQueries`, `TestStatusControllerSearch`) is comprehensive for the single-shard / GitHub-Apps-auth paths it covers, but does not exercise the >100-repo end-to-end reset path or the non-GitHub-Apps re-merge path.

## Open questions
- Is the `TEMP: default max_query_concurrency to 25` commit intended to be squashed/dropped before merge, or is 25 meant to become the permanent default?
- For non-GitHub-Apps auth, was the query re-merge in `status.go:650-662` intentional, or a leftover that should also shard per-org?
- Was the org-level watermark/reset logic in `search()` designed against a single-shard-per-org model and just not updated for the multi-shard case?
- Is sync-controller sharding planned as a follow-up, and should the PR title/description be narrowed to "status controller" in the meantime?
