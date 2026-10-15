---
pr: kubernetes-sigs/prow#1012
title: 'Disambiguate git checkout from pathspecs with trailing "--"'
head_sha: a50883b90135f5df57159ab213c1015797d06b41
base: main
reviewed_at: 2026-10-15T23:05:58Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-15T23:15:24Z
  gated_head_sha: a50883b90135f5df57159ab213c1015797d06b41
  reviewed_head_sha: a50883b90135f5df57159ab213c1015797d06b41
---
# Review

## Gate

Decision: merge.

The current PR head, `a50883b90135f5df57159ab213c1015797d06b41`, is unchanged from the reviewed head. The local review contains no actionable findings, and GitHub has no submitted reviews, inline comments, or substantive issue comments raising concerns. The compatibility pass found no unacceptable merge risk.

Gating list:

- None. There are no unresolved findings or confirmations required before merge.

Independent merge risk:

- No exported API or configuration changes.
- `pkg/git/v2/interactor.go:215` makes checkout arguments resolve as revisions when branch names also match working tree paths. Its scope includes v2 checkout consumers such as OWNERS loading, merge setup, pull request checkout, and rebase recovery; the existing shared helper applies the fix without migration or coordinated rollout.
- No notable merge risk was identified. The regression case mocks Git, so actual branch/path collision behavior is not exercised by that test; this remains a coverage limitation rather than a merge blocker.

## Verdict

Approve. No actionable defects found.

The change consistently updates the shared checkout helper and its callers' test expectations. Review of the complete PR diff, executor behavior, relevant callers, and existing tests found no concrete regression introduced by the change.

## What this PR does

- Adds a trailing `--` to the v2 interactor's `git checkout` invocation.
- Applies the change through the shared helper used by merge setup, pull request checkout, and rebase recovery.
- Updates command assertions and adds a path-like branch name regression case.

## Findings

No findings.

## Checked

- Reviewed the complete changes in `pkg/git/v2/interactor.go` and `pkg/git/v2/interactor_test.go`.
- Traced checkout calls through merge setup, pull request checkout, rebase recovery, and OWNERS loading.
- Inspected the executor implementation and updated fake-executor expectations.
- The new regression case verifies command arguments through a mock; it does not exercise the actual Git branch/path collision.
- Tests were inspected but not run.

## Open questions

None.
