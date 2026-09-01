---
issue: kubernetes-sigs/prow#729
title: "[Feature] Trigger required tests with /test-required"
state: open
labels: kind/feature, good first issue, help wanted
main_sha: 71428b9c282ee8c9e7e9512068fccce86e7915da
triaged_at: 2026-07-26T11:40:35Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 1
recommended_labels: [kind/feature, good-first-issue, help-wanted, area/plugins]
refresh_log:
  - at: 2026-07-26T11:40:35Z
    previous: 2026-06-15T16:38:16Z
    summary: "ranibharti385 self-assigned via /assign comment on 2026-07-25; no other activity."
---

## Findings

### [cause] No bulk command triggers explicit-trigger required presubmits not yet run
- detail: Jobs with `AlwaysRun: false` and no file-change pattern have `NeedsExplicitTrigger() = true`. Every existing bulk command excludes them: `TestAllFilter` returns `!p.NeedsExplicitTrigger()` (false → skip); `RetestFilter` "not yet run" clause is `!p.NeedsExplicitTrigger() && !allContexts.Has(p.Context)` (also skips). Only `/test <name>` works, requiring manual enumeration.
- evidence: `pkg/pjutil/filter.go:148,237`; `pkg/config/jobs.go:538-539`

### [cause] Prucek implementation sketch would not solve the use case if naively followed
- detail: Comment on issue says "new filter following `RetestRequiredFilter` pattern." But `RetestRequiredFilter` delegates to `RetestFilter` which uses `!p.NeedsExplicitTrigger()` — explicit-trigger jobs still skipped. The correct filter needs `forced = true` for `NeedsExplicitTrigger() = true` jobs so `Presubmit.ShouldRun` bypasses the explicit-trigger check via its `if forced { return true, nil }` branch.
- evidence: `pkg/pjutil/filter.go:255-260`; `pkg/config/jobs.go:525`

### [related-code] Regex definitions — TestRequiredRe goes here
- where: `pkg/pjutil/filter.go:30-37`
- excerpt: |
    var TestAllRe = regexp.MustCompile(`(?m)^/test all,?($|\s.*)`)
    var RetestRe = regexp.MustCompile(`(?m)^/retest\s*$`)
    var RetestRequiredRe = regexp.MustCompile(`(?m)^/retest-required\s*$`)

### [related-code] RetestRequiredFilter — direct model for TestRequiredFilter
- where: `pkg/pjutil/filter.go:244-263`
- excerpt: |
    func (rrf *RetestRequiredFilter) ShouldRun(ps config.Presubmit) (bool, bool, bool) {
        if ps.Optional { return false, false, false }
        return NewRetestFilter(rrf.failedContexts, rrf.allContexts).ShouldRun(ps)
    }

### [related-code] Presubmit.ShouldRun — forced=true bypass is the key leverage point
- where: `pkg/config/jobs.go:518-530`
- excerpt: |
    if ps.AlwaysRun { return true, nil }
    if forced       { return true, nil }   // ← bypasses NeedsExplicitTrigger jobs
    determined, shouldRun, err := ps.RegexpChangeMatcher.ShouldRun(changes)
    return (determined && shouldRun) || defaults, err

### [related-code] PresubmitFilter factory — new TestRequiredRe branch goes here
- where: `pkg/pjutil/filter.go:269-299`

### [related-code] commentMatchesTrigger — needs pjutil.TestRequiredRe added
- where: `pkg/plugins/trigger/generic-comment.go:217-230`

### [related-code] RetestLabel — must NOT be extended to /test-required
- where: `pkg/plugins/trigger/generic-comment.go:165-167`

## Checked
- `TestAllFilter.ShouldRun` returns `!p.NeedsExplicitTrigger(), false, false` — confirmed skips `AlwaysRun: false` + no-file-pattern jobs
- `RetestFilter.ShouldRun` "not yet run" clause requires `!NeedsExplicitTrigger()` — confirmed `/retest-required` also skips these jobs unless failed
- `Presubmit.ShouldRun` at `jobs.go:525`: `if forced { return true, nil }` — correct bypass for explicit-trigger jobs
- `RetestLabel` at `generic-comment.go:165` only fires on `/retest` and `/retest-required` — new command should not touch this
- Existing labels (`kind/feature`, `good-first-issue`, `help-wanted`) already applied by @Prucek

## Since previous triage
- 2026-07-25: ranibharti385 commented `/assign` and is now assigned to the issue. No design discussion, no PR opened yet.

## Next steps
- Post clarifying comment: filter must return `forced = true` for `NeedsExplicitTrigger() = true` jobs; without it the naive implementation won't work
- Confirm missing-only vs missing+failed behavior with author (author prefers missing-only — note it so contributor doesn't over-engineer)
- Consider adding `/area plugins` label
- ranibharti385 is now assigned — the clarifying comment above should be posted for their benefit if not already done

## Open questions
- Should `/test-required` cover only jobs with no prior context (missing-only) or also failed required jobs? Author leans missing-only — needs confirmation.
- Should required `run_if_changed` jobs be included when file changes don't match? Proposed filter returns `forced = false` for them (skipped by `Presubmit.ShouldRun` if files don't match) — is that correct?
