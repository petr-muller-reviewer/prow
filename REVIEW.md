---
pr: kubernetes-sigs/prow#735
title: "trigger: add /test-manual-required command"
head_sha: dedd4fc90801a7b2c23936a9c53f2cb6a42e2e4f
base: main
reviewed_at: 2026-09-02T00:25:25Z
verdict: request-changes
refresh_log:
  - from_sha: 11e9af6600623d7faec3c382e45e4c538ad66b7e
    to_sha: 11e9af6600623d7faec3c382e45e4c538ad66b7e
    at: 2026-06-01T15:50:53Z
    summary: "No code change. Maintainer Prucek posted inline review confirming NeedsExplicitTrigger() should be removed, and issue comment requesting help entry."
  - from_sha: 11e9af6600623d7faec3c382e45e4c538ad66b7e
    to_sha: 124f7da44fcbfd71622252c9d7459ecd1e267ee4
    at: 2026-06-15T12:44:06Z
    summary: "Author addressed earlier filter and help findings."
  - from_sha: 124f7da44fcbfd71622252c9d7459ecd1e267ee4
    to_sha: dedd4fc90801a7b2c23936a9c53f2cb6a42e2e4f
    at: 2026-09-02T00:25:25Z
    summary: "Author implemented the agreed rename and unconditional manual-required filter; one ContextRequired() edge case remains."
---

## Summary

Adds `/test-manual-required`, which starts manually triggered, non-optional presubmits without file-change conditions. The filter forces the selected jobs to run, and the command is wired into trigger matching and plugin help.

Since previous review:

- Renamed `/test-required` to `/test-manual-required` and removed the status-context lookup.
- Changed the filter to select explicitly triggered jobs unconditionally, excluding `always_run` and file-conditional jobs.
- Updated unit and trigger-plugin coverage. The PR is now merged.

## Findings

### [should-fix] Define “required” with `ContextRequired()`
- where: `pkg/pjutil/filter.go:276`
- concern: `!ps.Optional` still selects a `skip_report: true` manual job. Such a job publishes no GitHub context, so it cannot be a required presubmit context. Use `ps.ContextRequired()` and add a `skip_report: true` regression case.
- excerpt: |
    if ps.Optional || !ps.NeedsExplicitTrigger() || ps.RunIfChanged != "" || ps.SkipIfOnlyChanged != "" {
        return false, false, false
    }

### Resolved

#### [should-fix] Filter predicate should select manual jobs and force them
- Addressed in `dedd4fc90`: the filter now requires `NeedsExplicitTrigger()`, excludes change matchers, and returns `true, true, false`; it no longer fetches status contexts.

#### [should-fix] Command name was misleading
- Addressed in `dedd4fc90`: renamed to `/test-manual-required` in the matcher, documentation, and help command.

#### [nit] Tests needed to reflect the manual-only selection
- Addressed in `dedd4fc90`: tests cover required, optional, `always_run`, and conditional presubmits.

## Checked

- `NeedsExplicitTrigger()` is false for `always_run` and for either file-change matcher, so the explicit checks are redundant but behaviorally correct.
- Filter ordering, command recognition, help pruning, and help-provider wiring are correct.
- Diff from the saved baseline is 78 additions / 101 deletions across the five trigger and pjutil files; it is a focused update.

## Activity since previous review

- 2026-08-10: Amulyam24 pushed `dedd4fc90`, implementing the discussed design and rename.
- 2026-08-11 and 2026-08-13: petr-muller and Amulyam24 confirmed trusted-member behavior matches `/test`.
- 2026-08-19: petr-muller submitted an approval; the PR is now merged.

## Followups

Accepted: 2 (one deliverable, one probe). Skipped: 0.

### Exclude non-reporting jobs from the required manual selection

- Category: correctness
- Necessity: should
- Mode: deliver
- Where: `pkg/pjutil/filter.go:275`, `pkg/pjutil/filter_test.go`, `pkg/plugins/trigger/generic-comment_test.go`
- Why followup: the saved required-context finding remains in the merged implementation; `skip_report: true` manual jobs can be selected despite publishing no required GitHub context.

```text
In kubernetes-sigs/prow, following PR #735 — "trigger: add /test-manual-required command"
(merge commit 7593870dc2d79348c29ed4acaa2c249bd3058b32), work from the current
default branch containing that merge and first check whether this issue is already fixed.

Update TestManualRequiredFilter.ShouldRun in pkg/pjutil/filter.go to use
Presubmit.ContextRequired() when selecting required jobs. Add regression coverage
for an otherwise eligible manual job with Optional:false and SkipReport:true:
/test-manual-required must not select or start it. Preserve the selection of
reporting required manual jobs and the exclusion of optional, automatic, and
file-conditional jobs. Reuse the existing pjutil and trigger fixtures.

Acceptance criteria: the regression case demonstrates the original issue and
passes with the fix; existing relevant selections remain unchanged; run
go test ./pkg/pjutil ./pkg/plugins/trigger successfully.

Scope: only this command's required-job predicate and focused regression coverage.
Do not change /retest-required, Tide policy, trust handling, or unrelated filters.
Leave the result local for review; do not publish or open a PR.
```

### Probe trust handling and branch restrictions

- Category: tests
- Necessity: could
- Mode: probe
- Where: `pkg/plugins/trigger/generic-comment_test.go`, `pkg/pjutil/filter_test.go`
- Why followup: the PR discussion confirmed trust behavior, but its three new integration cases all use a trusted commenter; the new force-run path also merits exercising branch restrictions.

```text
In kubernetes-sigs/prow, following PR #735 — "trigger: add /test-manual-required command"
(merge commit 7593870dc2d79348c29ed4acaa2c249bd3058b32), work from the current
default branch containing that merge. Mode: probe.

Write temporary tests using the existing trigger and pjutil fixtures to exercise
/test-manual-required across these boundaries:
1. An untrusted commenter on an untrusted PR without ok-to-test cannot start jobs
   and receives the existing trust response.
2. A trusted commenter can start required manual jobs on that PR without granting
   persistent ok-to-test approval or removing needs-ok-to-test.
3. An untrusted commenter on a PR with ok-to-test can start eligible jobs.
4. A required manual job excluded by its branch filter does not run despite the
   command's forced=true selection.

Inspect the current implementation and fixtures before setting expectations. Run
the affected tests and classify each failure as an implementation bug, an incorrect
expectation, or a fixture mistake. A real bug means unauthorized execution, an
unintended persistent trust grant, failure to execute an eligible authorized job,
or execution of a job excluded by its branch filter.

Acceptance criteria: exercise every named case and explain any failures with
evidence. Apply only minimal production fixes for confirmed bugs. Remove all
temporary probe tests before finishing; if no bugs are found, leave no files
changed. After any production fix, run go test ./pkg/pjutil ./pkg/plugins/trigger.
Report the failing case for each confirmed bug so the user can choose whether to
retain a regression test separately.

Scope: no permanent test deliverable, test-only fixes, Tide policy changes, new
labels, help redesign, or unrelated cleanup. Preserve pre-existing user changes.
Leave any fixes local for review; do not publish or open a PR.
```
