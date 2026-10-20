---
pr: kubernetes-sigs/prow#792
title: "tide: update search API to ISSUE_ADVANCED"
head_sha: 4e28b64582617791d94b99ad452847cdd34588ac
base: main
reviewed_at: 2026-10-20T14:15:42Z
verdict: request-changes
---

# Review

## Verdict

Request changes. The PR selects a GraphQL search enum that GitHub scheduled for removal, and several query paths still join repository qualifiers with spaces. Those paths can fail against current schemas or stop returning results for valid multi-repo configurations.

## What this PR does

- Updates the `githubv4` dependency.
- Switches Tide, blocker, and PR status searches to `ISSUE_ADVANCED`.
- Shards Tide searches by organization regardless of authentication mode.
- Uses explicit `OR` between repositories in the dashboard's generated query.

## Findings

### [blocking] Use the supported GraphQL search type
- where: `pkg/tide/tide.go:2111`
- concern: GitHub's breaking-change documentation scheduled removal of `SearchType.ISSUE_ADVANCED` for November 4, 2025, because advanced search becomes the default for `ISSUE`. That date has passed, so schemas that applied the change can reject this query; the same enum is used in `pkg/tide/blockers/blockers.go:264` and `pkg/prstatus/prstatus.go:137`. Use `ISSUE` while keeping the advanced query syntax ([GitHub breaking changes](https://docs.github.com/en/enterprise-server@3.19/graphql/overview/breaking-changes), [API announcement](https://github.blog/changelog/2025-03-06-github-issues-projects-api-support-for-issues-advanced-search-and-more/)).
- excerpt: |
    } `graphql:"search(type: ISSUE_ADVANCED, first: 37, after: $searchCursor, query: $query)"`

### [should-fix] Keep same-org repository alternatives disjunctive
- where: `pkg/tide/tide.go:2111`
- concern: Tide query builders still emit multiple `repo:` qualifiers for one org separated by spaces. With advanced search, spaces mean `AND`, so queries for multiple configured repos in one org can return no PRs; the blocker and status searches share this query construction. Use explicit `OR` or separate searches, and cover a same-org multi-repo configuration in tests ([GitHub search syntax](https://github.blog/changelog/2025-03-06-github-issues-projects-api-support-for-issues-advanced-search-and-more/)).
- excerpt: |
    } `graphql:"search(type: ISSUE_ADVANCED, first: 37, after: $searchCursor, query: $query)"`

### [should-fix] Preserve repository OR in dashboard fallback queries
- where: `pkg/prstatus/prstatus.go:137`
- concern: When a custom dashboard query has no `org:` or `repo:` qualifier, `HandlePrStatus` appends configured repositories with spaces. Under advanced search, multiple repos become an `AND`, so the fallback can return no results. Use `OR` in this fallback and cover it with a multi-repo custom-query test.
- excerpt: |
    } `graphql:"search(type: ISSUE_ADVANCED, first: 100, after: $searchCursor, query: $query)"`

## Checked

- Tide, blocker, status, and dashboard query construction call sites.
- The documented removal schedule for `ISSUE_ADVANCED` and the new advanced-search qualifier semantics.
- The concern that the generated dashboard query's `OR` bypasses shared filters; GitHub's documented syntax shows shared filters before owner alternatives.
- Existing tests cover org sharding and the generated dashboard query, but not the same-org repo or custom-query fallback cases.
- No tests were run.

## Open questions

- Can tests cover multiple configured repositories in one org and an unscoped custom dashboard query?
