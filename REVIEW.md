---
pr: kubernetes-sigs/prow#961
title: "`peribolos`: use REST for direct collaborators to avoid GraphQL resource limits"
head_sha: 49e12d33e5523f175b2955490d52ceac0d12171e
base: main
reviewed_at: 2026-09-22T16:27:06Z
verdict: approve
---

## Verdict

Approve. The change avoids the GraphQL resource limit while preserving direct filtering, pagination, permission conversion, and fail-closed handling of pagination errors.

## What this PR does

- Replaces the GraphQL direct-collaborator query with the REST collaborators endpoint.
- Requests `affiliation=direct` and paginates at 100 entries per page.
- Reuses `LevelFromPermissions` for REST permission flags.
- Removes obsolete GraphQL code and adds focused REST tests.

## Findings

### Blocking

None.

### Should fix

None.

### Nits

None.

### Questions

None.

## Checked

- `pkg/github/client.go:4371-4391` uses the existing pagination helper, filters by `affiliation=direct`, and returns no partial result on an error.
- `pkg/github/client.go:4383-4385` uses `LevelFromPermissions`.
- `pkg/github/client_test.go:2410-2490` covers parameters, pagination, all standard permission levels, and a later-page error.
- `pkg/github/client.go:4352-4362` documents the REST/GraphQL semantic assumption and the GraphQL resource-limit motivation.
- Moving these reads to GitHub's REST/core rate-limit budget is an acceptable trade-off for the addressed GraphQL failure.

## Open questions

- No author response is needed. Monitor REST/core rate-limit use for representative large-org `--fix-collaborators` runs; add pacing or telemetry only if it proves material.
- `affiliation=direct` is an external GitHub API contract. The empirical validation recorded at `pkg/github/client.go:4352-4357` is sufficient here; an integration check could strengthen it later.
