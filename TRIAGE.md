---
issue: kubernetes-sigs/prow#729
title: "[Feature] Trigger required tests with /test-required"
state: closed
labels: kind/feature, good first issue, help wanted
main_sha: e2e19f37128f251b39be4e9ebf14ab52584e8dd5
triaged_at: 2026-09-02T15:08:31Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [kind/feature, good-first-issue, help-wanted, area/plugins]
---

## Findings

### [reproducibility] Required manual presubmits could only be started individually
- detail: Before the fix, `/test all` skipped explicit-trigger jobs and `/retest-required` only covered required failures. Users had to enumerate each pending job with `/test <job>`.
- evidence: `pkg/pjutil/filter.go:141-155,261-290`; issue report from lentzi90.

### [cause] No bulk filter selected manual required jobs
- detail: `NeedsExplicitTrigger()` identifies non-`always_run` jobs without file-match conditions. Existing bulk filters exclude them, so no command selected that job class.
- evidence: `pkg/config/jobs.go:510-539`; `pkg/pjutil/filter.go:141-155,261-290`.

### [related-code] Dedicated forced filter and trigger dispatch
- where: `pkg/pjutil/filter.go:294-346`
- excerpt: |
    if ps.Optional || !ps.NeedsExplicitTrigger() || ps.RunIfChanged != "" || ps.SkipIfOnlyChanged != "" {
        return false, false, false
    }
    return true, true, false
- relevance: `forced=true` makes `Presubmit.ShouldRun` start the selected job without file-condition evaluation.

### [related-code] Command recognition and help
- where: `pkg/plugins/trigger/generic-comment.go:217-280`; `pkg/plugins/trigger/trigger.go:147-153`
- detail: The trigger plugin recognizes `/test-manual-required`, builds the presubmit filter, and documents it as a featured command.

### [related-pr] PR #735 implemented the feature
- ref: kubernetes-sigs/prow#735
- relevance: Merged 2026-08-19; GitHub records it as closing this issue. It added `/test-manual-required` rather than the proposed `/test-required` name.

## Checked

- Issue #729 state, labels, comments, close event, and GitHub closing reference.
- PR #735 metadata and changed files.
- Current filter, trigger-dispatch, command-help, and focused test paths.
- `pkg/pjutil/filter_test.go:812-868` and `pkg/plugins/trigger/generic-comment_test.go:599-679` cover selection and exclusion cases.

## Next steps

- Keep closed as completed; no code or label action is needed.
- Treat an operational failure of `/test-manual-required` as a new bug report with presubmit configuration and command output.

## Open questions

- None. The implementation intentionally targets required, explicitly triggered jobs without file conditions; it is not a status-aware missing-only command.
