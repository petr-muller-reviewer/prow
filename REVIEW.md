---
pr: kubernetes-sigs/prow#1023
title: "ghcache: key coalesced requests by URL and Accept header"
head_sha: b2578bcd0ce3a1f9da4e22a933c8dfcc34c05a53
base: main
reviewed_at: 2026-10-19T11:06:15Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-22T00:08:39Z
  gated_head_sha: b2578bcd0ce3a1f9da4e22a933c8dfcc34c05a53
  reviewed_head_sha: b2578bcd0ce3a1f9da4e22a933c8dfcc34c05a53
---

# Review

## Gate

**Verdict: merge.**

The reviewed head is unchanged (`b2578bcd0ce3a1f9da4e22a933c8dfcc34c05a53`), and there are no prior reviewer findings, change requests, or substantive PR comments to disposition. The change is confined to the internal ghcache coalescing key, its regression test, and documentation, and introduces no API, configuration, persistence-format, or deployment-ordering changes. Its behavioral effect is the intended correction: same-URL GETs with different `Accept` values now make separate upstream requests instead of sharing the first response.

Gating list:

- None. `REVIEW.md` records no findings, and the PR has no non-pending reviews, inline comments, or substantive issue comments.

Area-2 risks:

- No notable merge risk. Existing deployments need no migration or coordinated rollout; the only affected callers are overlapping GETs for the same URL with different `Accept` values, which previously received the wrong representation.

## Verdict

Approve.

The implementation separates concurrent GET responses by URL and all `Accept` values, addressing issue #1020's representation mix-up. Code Quality and Maintainability both recommend approval; maintenance burden and deployment risk are low, with no actionable findings. The regression test explicitly overlaps requests and verifies their response bodies, and the targeted tests passed. A normal rolling deployment needs no configuration changes or migration; distinct representations now make separate upstream requests, so slightly lower coalescing is expected.

## What this PR does

- Adds the full `Accept` value list to the request coalescing key.
- Covers overlapping same-URL requests for JSON, diff, and patch representations.
- Updates ghcache documentation to describe the new coalescing condition.

## Findings

No actionable findings.

## Checked

- Compared the change with issue #1020; the implementation and test cover the requested behavior without unrelated scope.
- The request key includes every `Accept` field value, not only the first, while coalescing remains limited to GET requests.
- Reviewed the complete diff, the surrounding coalescer logic, cache construction, and GitHub client Accept header handling; no concrete regression was identified.
- Code Quality: the key includes every Accept value, and the test verifies overlapping upstream requests and each caller's response body. No improvements or critical issues identified.
- Maintainability: the change is proportional to the bug, preserves existing interfaces and synchronization, and explains the revised behavior in comments and documentation. No concerns or suggestions identified.
- Deployment Risk: configuration, persistent cache format, authorization partitions, throttling, and request cleanup remain compatible. No migration or deployment ordering requirements; older replicas retain the original bug until replaced.
- Passed `go test ./pkg/ghcache -run '^(TestRoundTripDifferentAccept|TestRoundTrip|TestCacheModeHeader)$' -count=1`.

## Open questions

None.

## Followups

### Make the persistent ghcache key Accept-aware

- Category: bugfix
- Necessity: should
- Mode: deliver

```text
In kubernetes-sigs/prow, following PR #1023 — "ghcache: key coalesced requests by URL and Accept header" at head b2578bcd0ce3a1f9da4e22a933c8dfcc34c05a53, fix the remaining representation-mixing path in ghcache's persistent cache.

PR #1023 makes the in-flight request coalescer key by URL and all Accept header values, but NewFromCache still uses github.com/cjwagner/httpcache.Transport, whose GET cache key is only req.URL.String(). As a result, sequential GETs for the same pull-request URL with JSON, diff, and patch Accept values can reuse the wrong cached response.

Implement an Accept-aware GET cache key for the full ghcache transport chain while preserving existing token-partition behavior. This will likely require a ghcache-owned transport or an upstream-compatible extension instead of directly relying on httpcache.Transport's URL-only key.

Acceptance criteria:
- Add a regression test through NewFromCache that exercises same-URL GETs with application/vnd.github.v3+json, application/vnd.github.v3.diff, and application/vnd.github.patch Accept values.
- Cover both sequential requests and overlapping requests, and verify each caller receives the response body corresponding to its Accept header.
- Verify distinct Accept representations are cached and revalidated separately under the same token partition.
- Run the focused ghcache tests and any broader package tests needed to validate the transport change.

Scope guard: do not change exported APIs, configuration, token-partition semantics, throttling, or unrelated coalescer behavior.
```
