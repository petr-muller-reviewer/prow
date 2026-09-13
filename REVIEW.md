---
pr: kubernetes-sigs/prow#930
title: "tide: exclude PR from subpool when its changed files cannot be fetched"
head_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
base: main
reviewed_at: 2026-09-13T20:00:21Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-09-13T20:00:41Z
  gated_head_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
  reviewed_head_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
refresh_log:
  - old_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
    new_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
    summary: "No code changes; incorporated maintainer/author discussion of PR-level visibility as a follow-up."
  - old_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
    new_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
    summary: "No code changes; incorporated approval of the separate PR-level visibility follow-up."
---

## Verdict

Approve. A changed-files lookup failure is now isolated to that PR instead of discarding its whole Tide subpool. The implementation removes partial presubmit state before excluding the PR, keeps unaffected PRs eligible, and has focused regression coverage.

## Gate

**Decision: merge.** The current head is the reviewed head (`0d14409f7124015f44dc25c4a29ce89f81407470`), so there are no post-review changes to reassess. `REVIEW.md` contains no blocking or should-fix findings. The only substantive discussion—PR-level visibility for an excluded PR—was explicitly accepted as a separate follow-up; it does not gate this narrowly scoped fix. The change has no API or configuration compatibility impact; its behavior deliberately isolates an unevaluable PR while preventing it from entering context checking, batch selection, or merge selection.

### Gating list

- None. No prior blocking or should-fix findings; PR-level visibility is explicitly deferred to a follow-up.

### Independent merge risk

- `pkg/tide/tide.go:1724-1734`: Tide will exclude a PR whenever changed-file evaluation errors, including transient non-422 errors. The affected PR cannot be batch-tested or merged by Tide, while other PRs in the subpool may proceed. This is an intentional, bounded availability tradeoff and matches the existing handling of per-PR presubmit-loading failures.
- No exported API, configuration schema/default, CLI, permission, or deployment-manifest change exists. Existing installations need no migration or release action.

## What this PR does

- Treats errors while evaluating changed-file-sensitive presubmits as per-PR failures.
- Removes any presubmits accumulated before that error and excludes the PR from the subpool.
- Defers adding a PR to the filtered subpool until all required-presubmit evaluation succeeds.
- Tests that another PR in the same subpool retains its required jobs and remains eligible.

Since previous review:

- No code changes; the PR head remains `0d14409f7124015f44dc25c4a29ce89f81407470`.
- @petr-muller asked how an indefinitely excluded PR should be made visible; @KR-Ravindra proposed a separate follow-up to expose the error through Tide status and the dashboard, or to fold it into this PR if preferred.

Since previous review:

- No code changes; the PR head remains `0d14409f7124015f44dc25c4a29ce89f81407470`.
- @petr-muller agreed to keep PR-level failure visibility as a focused follow-up and approved this PR on 2026-09-13.

## Findings

None.

## Checked

- `pkg/tide/tide.go:1724-1734`: partial presubmit results are deleted and the failed PR is absent from `sp.prs` before context checking, batching, or merge selection.
- `pkg/tide/tide.go:1695-1735`: the flow matches the existing per-PR handling of `GetPresubmits` failures and preserves the presubmits/subpool membership invariant.
- `pkg/tide/tide_test.go:3644-3691`: the regression test exercises a 422-style changed-files error after an `always_run` job, plus a second eligible PR.
- No configuration, API, schema, CLI, permission, or rollout changes are introduced.

## Open questions

None.
