---
pr: kubernetes-sigs/prow#1023
title: "ghcache: key coalesced requests by URL and Accept header"
head_sha: b2578bcd0ce3a1f9da4e22a933c8dfcc34c05a53
base: main
reviewed_at: 2026-10-19T11:06:15Z
verdict: approve
---

# Review

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
