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

## Followups

Accepted: 1 probe. Skipped: 0.

### Probe real-Git branch/path checkout behavior

- Category: tests
- Necessity: could
- Mode: probe
- Area: `pkg/git/git_test.go`, v2 checkout and secondary-clone fixtures.
- Why followup: the PR regression case checks mocked command arguments; real-Git coverage can exercise the reported collision without blocking the merge.

```text
In kubernetes-sigs/prow, perform a probe following PR #1012 — "Disambiguate git checkout from pathspecs with trailing \"--\"" (reviewed head a50883b90135f5df57159ab213c1015797d06b41). The PR was open when this followup was recorded; no merge commit was available. Work in a fresh worktree containing the PR's change, using the default branch after merge.

Mode: probe.

Write temporary real-Git integration tests around the v2 client's Checkout behavior. Reuse the localgit.NewV2 fixtures and patterns in pkg/git/git_test.go, pkg/git/localgit/localgit.go, and pkg/git/v2/client_factory.go; inspect pkg/git/v2/interactor.go and its mocked test in pkg/git/v2/interactor_test.go.

Exercise these behaviors:
- A fresh secondary clone has only a remote-tracking branch named team/app/overlays/prod, while that name also matches a tracked path. Give the target branch distinct content and verify that Checkout reaches the intended commit and updates the working tree. Explicitly establish that no matching local branch exists before the call.
- A matching local branch can be checked out even when its name matches a tracked path.
- A commit SHA can be checked out correctly, updating both HEAD and working-tree content.

Run the probes with the PR's fix present. Demonstrate that removing the trailing separator reproduces the original remote-branch/path ambiguity, then restore the implementation. That expected pre-fix failure is a control, not a newly discovered bug.

Classify each unexpected failure with evidence: real implementation bug, incorrect expectation, or fixture/test mistake. A real bug means a valid checkout fails, or reports success without selecting the intended commit and working tree, under the established fixture conditions.

Acceptance: exercise every named behavior and report the result, Git version, commands run, and evidence for each failure classification. If a real implementation bug is found, apply only its minimal fix and report the reproducer so the maintainer can decide whether to retain it as a regression test. If no bug is found, leave no files changed.

Scope: delete all probe tests and scratch artifacts before finishing. Restore any temporary implementation changes used for the pre-fix control. Do not submit the probe tests, fix unrelated code, refactor the Git client, or publish anything.
```
