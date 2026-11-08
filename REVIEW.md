---
pr: kubernetes-sigs/prow#930
title: "tide: exclude PR from subpool when its changed files cannot be fetched"
head_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
base: main
reviewed_at: 2026-11-08T00:04:27Z
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
  - old_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
    new_sha: 0d14409f7124015f44dc25c4a29ce89f81407470
    summary: "No code changes; recorded merge at 2026-09-13T20:17:17Z (13c2caa1) and the cross-referenced follow-up issue #937 / PR #938."
---

## Verdict

Approve. A changed-files lookup failure is now isolated to that PR instead of discarding its whole Tide subpool. The implementation removes partial presubmit state before excluding the PR, keeps unaffected PRs eligible, and has focused regression coverage.

## Gate

**Decision: merge.** The current head is the reviewed head (`0d14409f7124015f44dc25c4a29ce89f81407470`), so there are no post-review changes to reassess. `REVIEW.md` contains no blocking or should-fix findings. The only substantive discussion—PR-level visibility for an excluded PR—was explicitly accepted as a separate follow-up; it does not gate this narrowly scoped fix. The change has no API or configuration compatibility impact; its behavior deliberately isolates an unevaluable PR while preventing it from entering context checking, batch selection, or merge selection.

**Merged.** The PR merged 17 minutes after this review, at 2026-09-13T20:17:17Z, as `13c2caa1849f91c4186185eab709efc477feb185`, on the reviewed head with no intervening commits.

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

Since previous review:

- No code changes; the PR head remains `0d14409f7124015f44dc25c4a29ce89f81407470`.
- PR merged 2026-09-13T20:17:17Z as `13c2caa1849f91c4186185eab709efc477feb185`.
- The deferred visibility gap is tracked in kubernetes-sigs/prow#937 and addressed by kubernetes-sigs/prow#938 ("tide: report excluded PRs on their tide status context"), both still open on 2026-11-08.

## Findings

None.

## Checked

- `pkg/tide/tide.go:1724-1734`: partial presubmit results are deleted and the failed PR is absent from `sp.prs` before context checking, batching, or merge selection.
- `pkg/tide/tide.go:1695-1735`: the flow matches the existing per-PR handling of `GetPresubmits` failures and preserves the presubmits/subpool membership invariant.
- `pkg/tide/tide_test.go:3644-3691`: the regression test exercises a 422-style changed-files error after an `always_run` job, plus a second eligible PR.
- No configuration, API, schema, CLI, permission, or rollout changes are introduced.

## Open questions

None.

## Followups

Recorded by `/review:followup` on 2026-11-08, after the PR merged as `13c2caa1849f91c4186185eab709efc477feb185`. Two of five candidates accepted; the excluded-PR status follow-up (#937/#938), exclusion metrics/log-level alignment, and the dead `presubmitsByPull` error return were skipped.

### [should] deferred-review: `trigger` aborts when changed files are unfetchable

- where: `pkg/pjutil/filter.go:210-224`, `pkg/plugins/trigger/generic-comment.go:259-279`, `pkg/plugins/trigger/pull-request.go:422`
- mode: deliver
- why: deferred by this PR and tracked as kubernetes-sigs/prow#914; `/test all`, `/retest-required` and push-triggered selection still fail without feedback when GitHub answers 422 for an oversized diff.

```text
In kubernetes-sigs/prow, following post-merge followup to PR #930 ("tide: exclude PR from subpool when its changed files cannot be fetched", merged as 13c2caa1), make the `trigger` plugin's test selection survive a PR whose changed-file list GitHub cannot produce (issue #914).

Context: `pjutil.FilterPresubmits` (pkg/pjutil/filter.go:210-224) returns on the first `presubmit.ShouldRun` error. For `run_if_changed` / `skip_if_only_changed` presubmits, `ShouldRun` calls the lazy provider built by `config.NewGitHubDeferredChangedFilesProvider` (pkg/config/jobs.go:561-580), which calls `GetPullRequestChanges`. When GitHub answers 422 "Sorry, this diff is taking too long to generate", trigger fails `/test all`, `/retest-required` and push-triggered selection with no feedback. PR #930 fixed only the Tide side.

Task:
1. Wrap the `changes` provider passed into `pjutil.FilterPresubmits` so a fetch failure is memoized and recoverable instead of being lost inside the `ShouldRun` error.
2. When `presubmit.ShouldRun` errors AND the recorded changed-files fetch failed, treat that presubmit conservatively as "should run" (issue #914's expected behaviour), log a Warn containing the presubmit name and the underlying error, and continue evaluating the remaining presubmits.
3. Keep returning an error when `ShouldRun` fails but the recorded provider error is unset, so genuine programming errors are not swallowed.
4. Preserve laziness: do not call the provider when no presubmit in the filtered set needs the changed files.

Acceptance criteria:
- `go test ./pkg/pjutil/... ./pkg/plugins/trigger/...` passes.
- New unit tests in pkg/pjutil/filter_test.go prove: with a changes provider that always errors, a `run_if_changed` presubmit is included in the result and no error is returned; with a working provider and no matching files it is still excluded; `skip_if_only_changed` also falls back to running; the provider is invoked at most once per FilterPresubmits call.
- All existing FilterPresubmits and trigger tests keep their current behaviour.

Scope guard: do not touch pkg/plugins/approve (its file list feeds approver-set computation and needs a separate decision), Tide, the GitHub client, or the config schema. Do not add a configuration option, post PR comments, or change retry behaviour.
```

### [could] tests probe: exclusion edge cases in `presubmitsByPull`

- where: `pkg/tide/tide_test.go` (`TestPresubmitsByPull`), `pkg/tide/tide.go:1699-1734`
- mode: probe
- why: the new table case covers one failing PR whose failing `run_if_changed` job follows an `always_run` job; exclusion ordering, multiple failures, non-422 errors and recovery on the next sync are unexercised.

```text
In kubernetes-sigs/prow, following post-merge followup to PR #930 (merged as 13c2caa1), probe the per-PR exclusion path in `presubmitsByPull` (pkg/tide/tide.go:1699-1734).

Background: when `ps.ShouldRun(...)` errors - always a changed-files lookup failure, since `Presubmit.ShouldRun` only errors through `RegexpChangeMatcher.ShouldRun` -> the changes provider - the code stops evaluating that PR, warns, deletes any presubmits already accumulated for it, and skips appending it to `filteredPRs`/`sp.prs`.

This is a probe: write throwaway table cases in `TestPresubmitsByPull` (pkg/tide/tide_test.go) exercising these behaviours, run them, and triage every failure as a real implementation bug or a wrong expectation.

Behaviours to exercise:
1. Failing PR evaluated first (the table starts `sp.prs` with the case's PRs and appends the healthy samplePR #100): assert the failing PR has no entry in the returned presubmits map and only #100 remains in `sp.prs`.
2. Two failing PRs plus healthy #100: both excluded, #100 keeps every required presubmit.
3. Every PR in the subpool fails: assert the presubmits map is empty, `sp.prs` is empty, and nothing panics.
4. A non-422 error (for example "return code not 2XX: 500 Internal Server Error") behaves exactly like the 422-style error.
5. No poisoning across syncs: after a failing `presubmitsByPull` call, a second call whose changed-files lookup succeeds (or whose `changeCache` is pre-populated) re-admits the PR with its full presubmits - i.e. the failure was not cached.
6. If cheap to construct in this table: leftover ProwJobs of the excluded PR in `sp.pjs` do not affect the returned presubmits or `sp.prs`.

Acceptance criteria: every named behaviour is exercised; each failure is classified with evidence (input, expected, actual). If a real bug is found, apply only the minimal fix in pkg/tide/tide.go and report the failing case so it can be kept as a regression test. If none is found, change no implementation files.

Scope guard: delete the probe tests before finishing (the deliverable is a bug fix, never the tests); do not modify unrelated tests, Tide config, or the trigger plugin; do not commit.
```
