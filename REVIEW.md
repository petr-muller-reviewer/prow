---
pr: kubernetes-sigs/prow#961
title: "`peribolos`: use REST for direct collaborators to avoid GraphQL resource limits"
head_sha: 49e12d33e5523f175b2955490d52ceac0d12171e
base: main
reviewed_at: 2026-09-23T14:57:38Z
verdict: approve
---

# Review

## Verdict

**Approve with suggestions.** The REST implementation reuses the client's pagination, authentication, and permission mapping, and returns no partial collaborator set after a later-page error. No incompatible API or configuration change was found. The `affiliation=direct` contract is consequential for removals, but the reported live comparison with GraphQL found parity; preserving a repeatable validation and monitoring REST quota are non-blocking suggestions.

## What this PR does

- Replaces the GraphQL `DIRECT` collaborator query with the REST collaborators endpoint using `affiliation=direct`.
- Requests 100 collaborators per page through the existing REST pagination helper.
- Converts REST permission flags with `LevelFromPermissions` and removes the separate GraphQL mapper.
- Keeps the peribolos collaborator reconciliation interface and configuration unchanged.
- Adds tests for request parameters, multiple pages, all five standard permission levels, and a later-page error.

## Findings

### [question] Preserve validation of direct filtering
- where: `pkg/github/client.go:4353-4357`
- concern: Peribolos removes users returned by this method when they are absent from config (`cmd/peribolos/main.go:1305-1311`). GitHub's REST reference does not precisely establish that `affiliation=direct` excludes every inherited grant. The reported comparison with GraphQL on one 280-collaborator repository supports this change; could that validation be linked or made repeatable for team-only, organization-base-only, overlapping, and owner access? This is non-blocking without evidence of a mismatch.
- excerpt: |
    // meaning users with an explicit repository-level grant rather than access inherited through org or
    // team membership. The REST reference for affiliation=direct is loose (it cannot distinguish an
    // org-level grant from a repository-level one in the response), so this direct-only behaviour was
    // confirmed empirically: affiliation=direct returned the same set as the GraphQL
    // collaborators(affiliation: DIRECT) connection on a repository with 280 collaborators.

## Checked

- `pkg/github/client.go:4371-4391` sends `affiliation=direct`, sets `per_page=100`, uses the existing REST pagination path, and returns `nil, err` after a failed page.
- `pkg/github/client.go:4383-4385` uses `LevelFromPermissions`; `pkg/github/client_test.go:2410-2490` exercises all five standard levels with literal REST JSON, pagination, request parameters, and later-page failure.
- `cmd/peribolos/main.go:983-991` performs one collaborator read per configured repository when `--fix-collaborators` is enabled. The REST/core rate-limit budget and run duration should be watched during a representative large-org rollout.
- No exported method signature, peribolos flag, or configuration schema changes were found.
- The focused `pkg/github` test passed during this review. The `cmd/peribolos` package was still compiling when its test run was stopped; no result is claimed for it.

## Open questions

- Could the author link the REST–GraphQL comparison described at `pkg/github/client.go:4355-4357`, or document a repeatable check covering inherited and overlapping access? This is a suggestion, not a merge condition.
