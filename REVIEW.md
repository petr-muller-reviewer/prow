---
pr: kubernetes-sigs/prow#930
title: "tide: exclude PR from subpool when its changed files cannot be fetched"
head_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
base: main
reviewed_at: 2026-09-10T12:10:52Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-09-10T12:22:08Z
  gated_head_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
  reviewed_head_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
---

## Verdict

Approve. A changed-files lookup failure is now isolated to that PR instead of discarding its whole Tide subpool. The implementation removes partial presubmit state before excluding the PR, keeps unaffected PRs eligible, and has focused regression coverage.

## Gate

**Decision: merge.** The current head is the reviewed head (`0d14409f7124015f44dc25c4a29ce89f81407470`), so there are no post-review changes to reassess. `REVIEW.md` contains no blocking or should-fix findings, and there are no substantive human reviews, inline comments, or issue comments requiring disposition. The change has no API or configuration compatibility impact; its behavior deliberately isolates an unevaluable PR while preventing it from entering context checking, batch selection, or merge selection.

### Gating list

- None. No prior blocking or should-fix findings, and no substantive PR feedback remains unresolved.

### Independent merge risk

- `pkg/tide/tide.go:1724-1734`: Tide will exclude a PR whenever changed-file evaluation errors, including transient non-422 errors. The affected PR cannot be batch-tested or merged by Tide, while other PRs in the subpool may proceed. This is an intentional, bounded availability tradeoff and matches the existing handling of per-PR presubmit-loading failures.
- No exported API, configuration schema/default, CLI, permission, or deployment-manifest change exists. Existing installations need no migration or release action.

## What this PR does

- Treats errors while evaluating changed-file-sensitive presubmits as per-PR failures.
- Removes any presubmits accumulated before that error and excludes the PR from the subpool.
- Defers adding a PR to the filtered subpool until all required-presubmit evaluation succeeds.
- Tests that another PR in the same subpool retains its required jobs and remains eligible.

## Findings

None.

## Checked

- `pkg/tide/tide.go:1724-1734`: partial presubmit results are deleted and the failed PR is absent from `sp.prs` before context checking, batching, or merge selection.
- `pkg/tide/tide.go:1695-1735`: the flow matches the existing per-PR handling of `GetPresubmits` failures and preserves the presubmits/subpool membership invariant.
- `pkg/tide/tide_test.go:3644-3691`: the regression test exercises a 422-style changed-files error after an `always_run` job, plus a second eligible PR.
- No configuration, API, schema, CLI, permission, or rollout changes are introduced.

## Open questions

None.
