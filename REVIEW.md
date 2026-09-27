---
pr: kubernetes-sigs/prow#961
title: "`peribolos`: use REST for direct collaborators to avoid GraphQL resource limits"
head_sha: 49e12d33e5523f175b2955490d52ceac0d12171e
base: main
reviewed_at: 2026-09-27T14:30:57Z
verdict: approve
refresh_log:
  - old_sha: 49e12d33e5523f175b2955490d52ceac0d12171e
    new_sha: 49e12d33e5523f175b2955490d52ceac0d12171e
    summary: "No code changes; recorded approval and merge activity since the previous review."
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

Since previous review:

- No commits or code changes; the PR head remains `49e12d33e5523f175b2955490d52ceac0d12171e`.
- `petr-muller` commented that the PR looked good on 2026-09-23 at 15:01 UTC; `cblecker` approved it on 2026-09-24 at 05:09 UTC. The approval bot confirmed approval shortly afterward. No new inline review comments were added.
- The PR was merged by `app/kubernetes-prow` on 2026-09-24 at 05:30 UTC.

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

## Followups

### Fail an org dump when direct collaborators cannot be listed

- category: safety
- necessity: should — a partial dump can become a collaborator-removal plan if applied later.
- where: `cmd/peribolos/main.go:391-397`, `cmd/peribolos/main_test.go` (`fakeDumpClient` and dump tests).
- why followup: The dump already warned and continued on a collaborator-list error before this PR. PR #961 moved that read to REST, making it a useful time to close the existing failure path; it did not block the merge.
- prompt:

```text
In kubernetes-sigs/prow, following PR #961, "`peribolos`: use REST for direct collaborators to avoid GraphQL resource limits" (merged as a2505570f35fbde3ee46832d2253aa59fe803e1e), make peribolos fail an org dump when it cannot list a repository's direct collaborators.

In cmd/peribolos/main.go, dumpOrgConfig currently logs a warning for ListDirectCollaboratorsWithPermissions errors and still returns a config with that repository's collaborators omitted. A later --fix-collaborators run using the dumped config can remove those direct collaborators. Return a contextual error naming the organization and repository instead, so --dump and --dump-full do not print incomplete YAML. In cmd/peribolos/main_test.go, extend the dump fake to inject a collaborator-list failure and add a focused test that checks the error and absence of a partial config. Keep successful dumps, empty collaborator sets, and configureCollaborators behavior unchanged.

Acceptance criteria: a collaborator-list failure causes dumpOrgConfig to return an error with the affected org/repo and no config; both dump modes exit before emitting YAML through their existing error path; a successful empty collaborator list remains valid; the focused peribolos tests pass. Scope guard: do not change the REST client, collaborator reconciliation, or unrelated dump error handling.
```
