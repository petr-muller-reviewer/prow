---
pr: kubernetes-sigs/prow#869
title: "chore(deps): bump github.com/gomodule/redigo from 1.8.5 to 1.9.3"
head_sha: d4ee70a577e0d5090ae2369e490ef45ce060cf1d
base: main
reviewed_at: 2026-08-25T12:01:46Z
verdict: approve
refresh_log:
  - from: d4ee70a577e0d5090ae2369e490ef45ce060cf1d
    to: d4ee70a577e0d5090ae2369e490ef45ce060cf1d
    at: 2026-08-25T12:01:46Z
    summary: No code changes; no new comments/reviews. PR merged.
---

## What this PR does
- Bumps direct dependency `github.com/gomodule/redigo` from v1.8.5 to v1.9.3 in `go.mod`/`go.sum`.
- No project source code changes; dep-only bot PR (dependabot-style).
- Only importer in-tree: `pkg/ghcache/ghcache.go:47`, using `redis.Dial("tcp", redisAddress, ...)` with optional `DialUsername`/`DialPassword` (ghcache.go:543-559) for caching GitHub API responses. Single non-pooled connection; no context-aware redigo APIs used.
- Since previous review: no code changes and no new comments/reviews; PR merged.

## Findings

(none — dep-only bump, no project code changed)

## Checked
- Release freshness: v1.9.3 tagged 2025-10-08 (real tag, not pseudo-version), ~10.5 months old at review time — no soak-time concern.
- Full commit range v1.8.5..v1.9.3 (36 commits) and each intermediate release (v1.9.0-v1.9.3) changelog: no CVEs/security fixes. Substantive changes are additive context-aware APIs (`DialURLContext`, `DoContext`, `ReceiveContext`, `TestOnBorrowContext`), Valkey URL-schema support, a `ScanStruct` tweak, retraction of a bad v1.8.10 tag, plus CI/test/doc fixes.
- Confirmed `redis.Dial`, `DialUsername`, `DialPassword` (the only APIs this repo calls) have no breaking/behavioral changes across the range.
- Import surface: single file (`pkg/ghcache/ghcache.go`), light/non-sensitive-path exposure (GitHub API response cache, not auth/token/crypto core path).

## Open questions
(none)
