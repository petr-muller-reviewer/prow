---
pr: kubernetes-sigs/prow#982
title: "shrink page sizes on timeout and retry graphql errors"
head_sha: 8f28257f115b3ad6af8a591a3757720ae75cb1c8
base: main
reviewed_at: 2026-10-01T15:00:30Z
verdict: request-changes
---

# Review

## Verdict

Request changes pending clarification or protection of mutation retry safety.

All three reviewer perspectives raised the shared mutation retry policy. The code confirms that mutations inherit automatic retries; the scenario where GitHub commits a mutation before returning a gateway error is plausible, but was not demonstrated. The author should establish the safety guarantee or restrict retries to queries with explicit opt-in for safe mutations. No configuration migration is needed; operators should expect additional requests and longer waits during GitHub failures.

## What this PR does

- Makes Tide's GraphQL search page size configurable per request.
- Halves the page size after 502/504 errors and retries the same cursor.
- Retries transient GraphQL HTTP responses with request-body replay and cancellable backoff.

## Findings

### [question] Establish mutation retry safety
- where: `pkg/github/client.go:857-859`
- concern: CONFIRMED: the shared transport retries replayable GraphQL requests without distinguishing queries from mutations. PLAUSIBLE: if GitHub completes a mutation such as `transferIssue` before a gateway returns an error, replay could return an error after success or repeat a side effect; GitHub's behavior in that scenario was not verified. What guarantee makes these mutations safe to retry, or can retries be restricted to queries with explicit opt-in for safe mutations?
- excerpt: |
    resp, err := t.upstream.RoundTrip(req)
    if err != nil || !t.shouldRetry(r, resp.StatusCode) || retries >= t.maxRetries {
        return resp, err
    }

### [nit] Avoid duplicate headers on retries
- where: `pkg/github/client.go:853-854`
- concern: The first attempt passes the original request through `addHeaderTransport`, which appends preview headers and the user agent. Retries clone that modified request and append those headers again; consider cloning before every attempt or making header insertion idempotent. No resulting GitHub request failure was demonstrated.
- excerpt: |
    req = r.Clone(r.Context())
    req.Body = body

### [nit] Consider a typed gateway status error
- where: `pkg/tide/github.go:185-192`
- concern: Tide recognizes gateway timeouts by parsing the GraphQL library's formatted error text. A typed status error exposed by the GitHub client would provide a more stable contract; the existing integration test already guards the current format.
- excerpt: |
    var gatewayTimeoutRe = regexp.MustCompile(`non-200 OK status code: 50[24]\b`)

### [nit] Consider jitter for shared retries
- where: `pkg/github/client.go:872-875`
- concern: Deterministic exponential backoff can synchronize retry traffic across clients during a shared GitHub outage. Consider adding jitter while preserving bounded retries and context cancellation; this is an operational suggestion, not an observed outage caused by the change.
- excerpt: |
    if err := t.sleep(r.Context(), backoff); err != nil {
        return nil, err
    }
    backoff *= 2

## Checked

- Tide reduces the page size and retries the same cursor after gateway timeouts.
- The retry transport replays request bodies using `GetBody` and cancels during backoff when the request context ends.
- Search decodes into a fresh result value per attempt and keeps the reduced page size on later pages.
- No configuration schema, permissions, or deployment migration changes were identified.
- Added tests cover cursor reuse, shrinking, backoff, response body closure, and the real GraphQL client path. Tests were inspected but not run.

## Open questions

- Can GitHub return a gateway error after applying one of our mutations, and what makes replay safe in that case?
- Can the GraphQL wrapper mark operations so mutations require explicit opt-in for retries?
