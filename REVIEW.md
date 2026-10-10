---
pr: kubernetes-sigs/prow#938
title: "tide: report excluded PRs on their tide status context"
head_sha: aac8df64b8281e8830abf8a0e31339c6600f0c09
base: main
reviewed_at: 2026-10-07T16:23:52Z
verdict: request-changes
refresh_log:
  - old_sha: aac8df64b8281e8830abf8a0e31339c6600f0c09
    new_sha: aac8df64b8281e8830abf8a0e31339c6600f0c09
    summary: "No code changes; incorporated the author's request for review after tests were green."
gate:
  decision: do-not-merge
  gated_at: 2026-10-10T15:11:34Z
  gated_head_sha: aac8df64b8281e8830abf8a0e31339c6600f0c09
  reviewed_head_sha: aac8df64b8281e8830abf8a0e31339c6600f0c09
---

## Gate

**Verdict: do-not-merge.** The blocking finding “Preserve the GetPresubmits exclusion reason” remains unchanged at the current PR head. When `GetPresubmits` fails for a PR, `presubmitsByPull` drops it without adding an exclusion reason, so status reporting can repeat the same failure instead of publishing the explanatory Tide status this PR is intended to provide.

### Gating list

- **`REVIEW.md` — Preserve the GetPresubmits exclusion reason (`pkg/tide/tide.go:1717-1720`):** still not addressed at head `aac8df64b8281e8830abf8a0e31339c6600f0c09`. Record the failure through `excludePR` and test the status path before merging.

### Independent merge risk

- No exported API, configuration schema, flags, or storage formats change. Internal Tide status handling changes for affected PRs: merge-requirement/context-policy failures now produce an error status, which can affect automation that gates on the `tide` context. This is intentional and limited to PRs encountering these failures; no migration or coordinated rollout is required.

## What this PR does

- Isolates per-PR merge-requirement failures instead of dropping an entire Tide subpool.
- Carries exclusion reasons to Tide status reporting so affected PRs get an explanatory error context.
- Since previous review: the author requested review after tests were green; no commits, inline comments, or submitted reviews were added.

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
