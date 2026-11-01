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

## Dependency followups

Mined `github.com/gomodule/redigo` 1.8.5 → 1.9.3 (direct; the `go.mod`/`go.sum` diff changes no other module — no transitive bumps). Range read: 36 commits across v1.8.5..v1.9.3, including intermediate releases v1.8.6–v1.9.2 and the release notes for v1.9.0, v1.9.1, v1.9.2 and v1.9.3. One accepted item.

### api-upgrade: bound ghproxy's Redis dial with a context
- module: `github.com/gomodule/redigo` 1.8.5 → 1.9.3
- category: api-upgrade
- where: `pkg/ghcache/ghcache.go:547-563` (`redis.Dial` at 559, options at 547-557, `logrus.Fatal` at 561-563); production caller `cmd/ghproxy/ghproxy.go:227`
- necessity: could — `DialContext`/`DialConnectTimeout` already existed in v1.8.5, so the unbounded dial predates this bump; the bump adds `DialURLContext` and makes `DialContext` run the AUTH/SELECT handshake under the context (fix #620), which makes the fix cheap at this call site.
- what the bump offers: #574 added `DialURLContext` — "Add DialURLContext so that we consumers have control over cancelation and timeout … which should be preferred to ensure requests can't hang forever"; #620 "respect ctx in cmds run in DialContext".
- proposed scope: thread a `context.Context` into `ghcache.NewRedisCache`, dial with `redis.DialContext` (keeping `DialUsername`/`DialPassword` and the secret-file read), bound it with a deadline when the caller has none, and add a test that a cancelled context fails fast.

```text
In kubernetes-sigs/prow, now that PR #869 (merged as 92a66380274c9205b3da783bfbe9fd3b3c70fe03) bumped github.com/gomodule/redigo to v1.9.3, make the ghproxy Redis cache connection fail fast instead of blocking forever.

What and where:
- pkg/ghcache/ghcache.go, NewRedisCache (line 541): dials with redis.Dial("tcp", redisAddress, dialOpts...) at line 559; options are built at lines 547-557 (redis.DialUsername, redis.DialPassword); a dial error is fatal via logrus.Fatal (561-563). No connect deadline today.
- cmd/ghproxy/ghproxy.go:227 is the only production caller, inside proxy(o *options, ...), which has no context.Context parameter today. main() has interrupts.WaitForGracefulShutdown() available (line 207).
- redigo v1.9.3, reached by this bump, added redis.DialURLContext(ctx, rawurl, options...) upstream commit "feat: implement DialURLContext (#574)" whose docs say it "should be preferred to ensure requests can't hang forever", and "fix: respect ctx in cmds run in DialContext (#620)" so AUTH/SELECT now honour the context. redis.DialContext(ctx, network, address, options...) already existed before the bump and is the lower-risk choice here because --redis-address stays host:port.

Task:
- Add a ctx context.Context parameter to ghcache.NewRedisCache and replace redis.Dial("tcp", redisAddress, dialOpts...) with redis.DialContext(ctx, "tcp", redisAddress, dialOpts...). Use redis.DialURLContext only if you also build a redis:// / rediss:// URL from --redis-address/--redis-username/--redis-secret-file; DialContext is preferred so the existing host:port flag keeps its meaning.
- Bound the dial: if the incoming context has no deadline, apply one (e.g. context.WithTimeout with a small, explicit constant or an existing ghproxy timeout flag), and cancel it once the connection is established.
- Thread a context from cmd/ghproxy/main -> proxy(...) -> NewRedisCache, cancelled by interrupts.WaitForGracefulShutdown, and update every other caller and test of NewRedisCache.
- Keep the DialUsername/DialPassword handling and the secret-file read unchanged.

Acceptance criteria:
- NewRedisCache with a cancelled or already-expired context against an unreachable address returns an error promptly (no multi-second hang); the error path is the same logrus.Fatal as today, just reachable.
- go build ./... and go test ./pkg/ghcache/... ./cmd/ghproxy/... pass.
- One focused commit; no unrelated refactors.

Scope guard:
- Do not switch ghproxy to a redis.Pool, do not touch github.com/cjwagner/httpcache or its redis cache, and do not add retry/backoff/reconnect logic - this is only about bounding the dial.
- Do not change --redis-address semantics or add new flags, and do not touch unrelated ghcache behaviour (coalescing, throttling, partitioning).
- Leave REVIEW.md / REVIEW.html alone.
```

Changelog surface examined and not turned into items (no call site of ours): `DoContext`/`ReceiveContext` (#537) and `TestOnBorrowContext` (#660) — commands are issued by `github.com/cjwagner/httpcache/redis` via `NewWithClient` (ghcache.go:564), we never call `Do`/`Send`/`Receive` and never use a pool; `ScanStruct` simple-string keys (#692), `Float64Map` (#605), latency parsing (#614), `ArgsFlat`/`ScanStruct` anonymous-pointer fix (#622), slice/map error surfacing (#580) — helpers we don't call; Valkey URL-schema support (#689) — `DialURL`-only; `redis.Dial`/`DialUsername`/`DialPassword` have no deprecations or behaviour changes in the range.
