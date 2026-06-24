---
pr: kubernetes-sigs/prow#754
title: "Fix pagination prefix duplication on GHE with redirect"
head_sha: 55cf6c6a64cb5f32d03ae849685a83076f5db4a3
base: main
reviewed_at: 2026-06-24T12:31:41Z
verdict: approve
---

## Findings

None.

## Checked

- Host matching correctness: `url.Parse` on `c.bases` entries yields expected `Host`/`Path`. Empty path for github.com, `/api/v3` for GHE. `TrimPrefix` behaves correctly in both cases.
- Fallback when no base matches: prefix stays empty, `TrimPrefix` is no-op, full `RequestURI` used. Not a regression from old code.
- `url.Parse` failures on bases silently skipped via `err == nil` guard. Appropriate since invalid bases fail at request construction.
- First-match semantics correct: `c.bases` entries are distinct hosts for failover, not same-host-different-path.
- Test matrix complete: non-GHE+redirect (existing), GHE without redirect (existing), GHE+redirect (new test).
- No caller impact: function signature unchanged, change internal to pagination loop.
- Port normalization mismatch theoretically possible but practically impossible.

## Open questions

None.
