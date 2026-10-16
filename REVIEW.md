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
