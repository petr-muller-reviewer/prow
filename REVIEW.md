---
pr: kubernetes-sigs/prow#792
title: "tide: update search API to ISSUE_ADVANCED"
head_sha: d138f93c72a130b13efa4ffe278fdbeb0267e34e
base: main
reviewed_at: 2026-10-20T12:43:13Z
verdict: request-changes
---

# Review

## Verdict

Request changes. Switching searches to `ISSUE_ADVANCED` changes how space-separated repository qualifiers are interpreted, but Tide and the dashboard fallback still emit multiple repo qualifiers joined by spaces. Those paths can stop finding results for configurations that include multiple repositories.

## What this PR does

- Updates the `githubv4` dependency.
- Moves Tide, blocker, and PR status GraphQL searches to `ISSUE_ADVANCED`.
- Shards Tide and blocker searches by organization.
- Joins repositories with explicit `OR` in the dashboard's generated query.

## Findings

### [should-fix] Keep same-org repo filters disjunctive
- where: `pkg/tide/tide.go:2111`
- concern: Tide query builders still emit multiple `repo:` qualifiers for one org separated by spaces. With `ISSUE_ADVANCED`, spaces mean `AND`, so a query for multiple configured repos in one org can match none of them; the blocker and status searches share this query construction. GitHub documents spaces as `AND` and `OR` for alternatives ([search syntax](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/filtering-and-searching-issues-and-pull-requests)).
- excerpt: |
    } `graphql:"search(type: ISSUE_ADVANCED, first: 37, after: $searchCursor, query: $query)"`

### [should-fix] Preserve OR in dashboard fallback repo scoping
- where: `pkg/prstatus/prstatus.go:137`
- concern: When a custom dashboard query has no `org:` or `repo:` qualifier, `HandlePrStatus` appends configured repositories with spaces. Under `ISSUE_ADVANCED`, this requires a result to match every configured repo, so the fallback returns no results when more than one repo is configured.
- excerpt: |
    } `graphql:"search(type: ISSUE_ADVANCED, first: 100, after: $searchCursor, query: $query)"`

## Checked

- The query construction call sites in Tide, blockers, status search, and the dashboard.
- Existing tests cover org sharding and the generated dashboard query, but not these multi-repo cases.
- No tests were run.

## Open questions

- Can tests cover multiple configured repositories in the same org and an unscoped custom dashboard query?
