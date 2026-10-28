---
issue: kubernetes-sigs/prow#482
title: "Explore GitHub's Improved Search API"
state: open
labels: kind/feature, area/tide, lifecycle/stale
main_sha: 3617afe983fa4df61a494fc2a60300367372de3a
triaged_at: 2026-10-28T00:47:58Z
verdict: accepted
advice:
  advised_at: 2026-10-28T01:20:54Z
  based_on_triaged_at: 2026-10-28T00:47:58Z
---

# Triage

## Verdict

**Accepted.** This is a legitimate exploratory feature request for Prow's Tide and GitHub search clients. Keep it open while the assigned investigator narrows it to a concrete Tide query or a measured API-call reduction opportunity. A future implementation is tentatively Level 3 (large): it affects merge-candidate search and has unresolved GitHub query semantics and result-limit constraints.

## Resolution

Not resolved; the issue remains open.

## What the issue reports

- GitHub added Boolean and nested issue-search expressions and the issue asks whether Prow can benefit.
- The main areas named are Tide query construction and several other Prow GraphQL/REST search callers.
- The suggested directions are more expressive Tide configuration or combining searches to make fewer API requests.
- No representative query, expected user workflow, or success measure is specified.

## Re-triage recommended

Not assessed by `/triage:refresh`.

## Findings

### [cause] Tide has no general nested Boolean configuration
- detail: `TideQuery` builds a flat set of qualifiers, with comma-separated alternatives for labels. It cannot express arbitrary nested Boolean groups. The REST search wrapper also does not opt into advanced search. No defect is reported; this is an exploratory feature gap.
- evidence: `pkg/config/tide.go:660-758`; `pkg/github/client.go:3905-3928`. GitHub's [REST search API](https://docs.github.com/en/rest/search/search?apiVersion=2022-11-28) documents `advanced_search=true` as the opt-in. GitHub's [GraphQL schema changelog](https://docs.github.com/en/graphql/overview/changelog/2025) scheduled removal of `SearchType.ISSUE_ADVANCED` effective 2025-11-04; verify current `ISSUE` behavior before relying on advanced syntax.

### [related-code] Tide query configuration and generation
- where: `pkg/config/tide.go:660-676,730-757`
- excerpt: |
    type TideQuery struct {
    	Author string `json:"author,omitempty"`
    
    	Labels        []string `json:"labels,omitempty"`
    	MissingLabels []string `json:"missingLabels,omitempty"`
    
    	ExcludedBranches []string `json:"excludedBranches,omitempty"`
    	IncludedBranches []string `json:"includedBranches,omitempty"`
    
    	Milestone string `json:"milestone,omitempty"`
    
    	ReviewApprovedRequired bool `json:"reviewApprovedRequired,omitempty"`
    
    	Orgs          []string `json:"orgs,omitempty"`
    	Repos         []string `json:"repos,omitempty"`
    	ExcludedRepos []string `json:"excludedRepos,omitempty"`
    }
    
    queryString := []string{"is:pr", "state:open", "archived:false"}
    for _, l := range tq.Labels {
    	var orOperands []string
    	for alt := range strings.SplitSeq(l, ",") {
    		orOperands = append(orOperands, fmt.Sprintf("\"%s\"", alt))
    	}
    	queryString = append(queryString, fmt.Sprintf("label:%s", strings.Join(orOperands, ",")))
    }
    return orgScopedIdentifiers, strings.Join(queryString, " ")

### [related-code] Tide runs separate GraphQL searches
- where: `pkg/tide/github.go:116-130`; `pkg/tide/tide.go:2180-2191`
- excerpt: |
    for i, query := range gi.cfg().Tide.Queries {
    	var queries map[string]string
    	if gi.usesGitHubAppsAuth {
    		queries = query.OrgQueries()
    	} else {
    		queries = map[string]string{"": query.Query()}
    	}
    
    	for org, q := range queries {
    		org, q, i := org, q, i
    		g.Go(func() error {
    			results, err := gi.search(gi.ghc.QueryWithGitHubAppsSupport, gi.logger, q, time.Time{}, time.Now(), org)
    
    } `graphql:"search(type: ISSUE, first: $searchPageSize, after: $searchCursor, query: $query)"`
- detail: Results are unioned and deduplicated. GitHub's 1,000-result search cap makes batching a correctness concern. Tests cover paging and resource-limit handling, but not Boolean grouping or safe batching.

### [related-code] REST search wrapper omits advanced-search options
- where: `pkg/github/client.go:3905-3928`
- excerpt: |
    values := url.Values{
    	"per_page": []string{"100"},
    	"q":        []string{query},
    }
    
    err := c.readPaginatedResultsWithValues(
    	"/search/issues",
    	values,
    	acceptNone,
    	org,
- detail: The direct production caller found is generic-autobumper, which currently builds a flat query (`cmd/generic-autobumper/updater/updater.go:70`). Other GraphQL search paths include blockers, needs-rebase, and the PR status dashboard.

### [cause] Semantic search is not suitable for merge eligibility
- detail: GitHub's semantic and hybrid issue-search API is intended for natural-language discovery and is rate limited to 10 requests per minute. Tide needs exact, repeatable predicate matching, so these modes should not drive merge decisions.
- evidence: [GitHub's April 2026 announcement](https://github.blog/changelog/2026-04-02-improved-search-for-github-issues-is-now-generally-available/) and the [REST search API docs](https://docs.github.com/en/rest/search/search?apiVersion=2022-11-28).

### [related-issue] Tide branch matching request
- ref: kubernetes-sigs/prow#438
- relevance: Open request for regex matching of Tide branch names; a separate feature, not a duplicate.

### [related-pr] OpenShift Tide configuration cleanup
- ref: openshift/release#81290
- relevance: Open PR changes OpenShift deployment configuration; it does not change Prow's search clients or API behavior.

## Resolved

None.

## Checked

- Read the issue, labels, assignee, comments, and timeline cross-references. It is open and assigned to Amulyam24; it has an automatic `lifecycle/stale` label. The assignee volunteered to investigate and share findings in April 2026.
- Searched Go sources for GraphQL `search(type: ...)`, REST `/search/issues`, and `FindIssues` call sites. Reviewed Tide query generation, query execution, pagination/error handling, and current tests.
- Existing tests cover Tide query generation and org sharding (`pkg/config/tide_test.go:459-491`), blocker query construction (`pkg/tide/blockers/blockers_test.go:78`), Tide search paging/resource limits (`pkg/tide/github_test.go:45,966-1087`), and REST search request/pagination (`pkg/github/client_test.go:2380-2504`). They do not verify nested Boolean matching or batching completeness.
- Searched for duplicates with `search API`, `GitHub search`, and `improved search`; no duplicate found. Issue #438 is distinct. No Prow PR implementing this feature was found; openshift/release#81290 only changes deployment configuration.
- Reviewed GitHub's advanced-search API announcement, current REST search and filter documentation, GraphQL schema changelog, and semantic-search release announcement.

## Next steps

- Ask assignee Amulyam24 to share the investigation and name one concrete Tide query that current config cannot express.
- Preserve the existing `TideQuery` behavior. If a concrete grouped rule exists, evaluate an additive typed Boolean-expression field and verify it against current GraphQL `ISSUE` behavior.
- If the goal is fewer API calls, measure real query counts and result sizes first; keep org sharding and the 1,000-result cap in view.
- Existing labels `kind/feature` and `area/tide` fit the request. No priority or newcomer label is supported by current evidence.

## Advice
1. Add `priority/awaiting-more-evidence`. The issue is a valid feature request, but it has no concrete Tide query or success measure yet. This label exists in the repository and is not currently applied.

```sh
gh issue edit 482 --repo kubernetes-sigs/prow --add-label "priority/awaiting-more-evidence"
```

2. If you want to resume the investigation, ping the assignee with a specific request and mark the issue fresh. The issue remains open and assigned to Amulyam24, but has had no new comments since the triage; `lifecycle/stale` is currently applied.

```sh
gh issue comment 482 --repo kubernetes-sigs/prow --body "> *This was generated by AI during triage.*

@Amulyam24, could you share an update with one Tide query that is awkward with the current TideQuery configuration, the Boolean expression you want to express, and whether the goal is simpler configuration, fewer API calls, or both? If exploring batching, please include representative result volumes given GitHub's 1,000-result cap.

/remove-lifecycle stale"
```

The existing `kind/feature` and `area/tide` labels are appropriate and already applied. The issue has no milestone, and the triage provides no basis for assigning one. The open cross-referenced PR `openshift/release#81290` changes deployment configuration and does not resolve the Prow search work.
## Open questions

- Could you share one Tide query that is awkward with the current configuration, along with the Boolean expression you want it to represent?
- Is the primary goal simpler Tide configuration, fewer GitHub requests, or both?
- If the goal is batching, what result volumes must remain complete when a combined query approaches GitHub's 1,000-result cap?
