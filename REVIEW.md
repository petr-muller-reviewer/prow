---
pr: kubernetes-sigs/prow#997
title: "tide: shrink search pages on resource limits error on 200s"
head_sha: 930bf73c82e7034df83513274e2d3225af527e09
base: main
reviewed_at: 2026-10-09T10:27:08Z
verdict: approve
---

# Review

## Verdict

Approve with suggestions. The change handles GraphQL resource-limit errors returned with HTTP 200 by retrying the current page at a smaller size, while preserving earlier successful pages and discarding partial data from the failed request. The maintainer reviewers found no blocking correctness or deployment issues; the main follow-up is replacing the acknowledged error-string match with structured errors when available.

## What this PR does

- Recognizes GitHub's `Resource limits for this query exceeded` GraphQL error message.
- Retries the current search cursor with a smaller page, down to the existing minimum size.
- Adds coverage for first-page and later-page retries, persistent failures, rate-limit errors, and HTTP 200 responses with partial data.

## Findings

### [should-fix] Replace error-message matching with structured error detection
- where: `pkg/tide/github.go:282-283`
- concern: The retry depends on an exact phrase in the GraphQL error message, so a wording change could prevent Tide from shrinking pages. This limitation is acknowledged in the PR; replace the string match when the client exposes structured resource-limit errors.
- excerpt: |
    func isSearchResourceLimit(err error) bool {
        return err != nil && strings.Contains(err.Error(), "Resource limits for this query exceeded")
    }

## Checked

- Code quality review found no actionable correctness issues; maintainability burden and deployment risk were assessed as low.
- A resource-limit error retries the same cursor with a smaller page; partial response data is discarded and prior successful results are retained.
- The change adds no configuration or API changes and requires no special upgrade steps.
- Tests cover the HTTP 200 error path and retry cases; they were inspected but not run during this review.
- Deployment reviewers recommend monitoring Tide search errors and request volume for affected searches.

## Open questions

None.
