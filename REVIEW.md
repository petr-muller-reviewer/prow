---
pr: kubernetes-sigs/prow#938
title: "tide: report excluded PRs on their tide status context"
head_sha: aac8df64b8281e8830abf8a0e31339c6600f0c09
base: main
reviewed_at: 2026-09-14T22:47:50Z
verdict: request-changes
---

## Findings

### [blocking] Preserve the GetPresubmits exclusion reason
- where: `pkg/tide/tide.go:1717-1720`
- concern: A per-PR `GetPresubmits` error still removes the PR from the subpool without calling `excludePR`. The status controller subsequently rebuilds the context policy, which may invoke `GetPresubmits` again and fail before setting the promised explanatory `tide` status. Record the reason in `sp.excluded` and cover the complete status-reporting path with a test.
- excerpt: |
    presubmitsForPull, err := c.provider.GetPresubmits(...)
    if err != nil {
        log.WithError(err).Debug("Failed to get presubmits for PR, excluding from subpool")
        continue
    }

## Checked

- Context-policy failures are isolated to the affected PR, leaving valid sibling PRs in the subpool.
- Exclusion state is collected from raw subpools, so an all-excluded subpool still reaches status reporting.
- Exclusion handling precedes context-checker construction in `expectedStatus`; merge-conflict status remains higher priority.
- The change adds no configuration schema, migration, permission, or external-dependency change.
- New unit tests cover changed-file failures, context-policy failures, dropped subpools, and conflict precedence.

## Open questions

- Could `GetPresubmits` failures be recorded through `excludePR` alongside changed-files and context-policy failures, with a status assertion that exercises the full `Sync()` handoff?
