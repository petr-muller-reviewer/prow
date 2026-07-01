---
issue: kubernetes-sigs/prow#777
title: "tide: context checker error silently drops PR instead of reporting via status context"
state: open
labels:
main_sha: 444074c659e1d2626895fbc542f7cc6ccfb19e16
triaged_at: 2026-07-01T22:19:04Z
verdict: accepted
---

## Findings

### [cause] Context list collision in GetTideContextPolicy (hypothesized)
- detail: When `from-branch-protection: true` is set, `GetTideContextPolicy` inserts BP's `RequiredStatusChecks.Contexts` into the `required` set (line 968). Separately, `BranchRequirements()` partitions presubmit jobs by trigger behavior — a conditionally-triggered, non-optional job lands in `requiredIfPresent` (line 627). If the same context appears in both sources, `Validate()` rejects it. This trigger is hypothesized based on the observed config but not yet confirmed as the exact culprit.
- evidence: `pkg/config/tide.go:962-976`, `pkg/config/branch_protection.go:604-638`

### [cause] Silent subpool drop on initSubpoolData error (confirmed)
- detail: `filterSubpools` logs the error and silently drops the entire subpool. No tide status context is set on any PR in the subpool. The status controller may show a misleading "missing requirements" status or nothing at all.
- evidence: `pkg/tide/tide.go:673-676`

### [related-code] GetTideContextPolicy — policy construction from BP and job properties
- where: `pkg/config/tide.go:944-995`
- excerpt: |
    if bp.Protect != nil && *bp.Protect && bp.RequiredStatusChecks != nil {
        required.Insert(bp.RequiredStatusChecks.Contexts...)
    }
    // ...
    prowRequired, prowRequiredIfPresent, prowOptional := BranchRequirements(branch, presubmits, requireManuallyTriggeredJobs)
    required.Insert(prowRequired...)
    requiredIfPresent.Insert(prowRequiredIfPresent...)

### [related-code] TideContextPolicy.Validate — rejects overlapping context lists
- where: `pkg/config/tide.go:862-891`
- excerpt: |
    if inter := sets.New[string](cp.RequiredContexts...).Intersection(sets.New[string](cp.RequiredIfPresentContexts...)); inter.Len() > 0 {
        return fmt.Errorf("contexts %s are defined as required and required if present", strings.Join(sets.List(inter), ", "))
    }

### [related-code] BranchRequirements — partitions jobs into required/requiredIfPresent/optional
- where: `pkg/config/branch_protection.go:604-638`
- excerpt: |
    case j.TriggersConditionally():
        requiredIfPresent = append(requiredIfPresent, j.Context)

### [related-code] TriggersConditionally and ContextRequired — job categorization
- where: `pkg/config/jobs.go:533-550`
- excerpt: |
    func (ps Presubmit) TriggersConditionally() bool {
        return ps.NeedsExplicitTrigger() || ps.RegexpChangeMatcher.CouldRun()
    }
    func (ps Presubmit) ContextRequired() bool {
        return !ps.Optional && !ps.SkipReport
    }

### [related-code] filterSubpools — silent error drop
- where: `pkg/tide/tide.go:665-690`
- excerpt: |
    if err := c.initSubpoolData(sp); err != nil {
        sp.log.WithError(err).Error("Error initializing subpool.")
        return
    }

### [related-code] initSubpoolData — wraps Validate error into subpool init failure
- where: `pkg/tide/tide.go:692-722`
- excerpt: |
    sp.cc[pr.Number], err = c.provider.GetTideContextPolicy(sp.org, sp.repo, sp.branch, refGetterFactory(string(sp.sha)), &pr)
    if err != nil {
        return fmt.Errorf("error setting up context checker for pr %d: %w", pr.Number, err)
    }

### [related-code] expectedStatus — existing error-reporting patterns in status controller
- where: `pkg/tide/status.go:277-368`
- excerpt: |
    // Line 293: StatusError for merge conflicts
    // Line 313: StatusError for blocking issues
    // Line 348: StatusPending for missing requirements

### [related-code] validateTideContextPolicy — static config validation (misses runtime collisions)
- where: `cmd/checkconfig/main.go:1183-1227`

## Checked

- Confirmed the error message in the issue matches `pkg/config/tide.go:869`
- Confirmed `checkconfig` validates context policies at config-load time but cannot catch runtime collisions from `from-branch-protection: true`
- Verified the status controller already uses `github.StatusError` for merge conflicts and blockers — same pattern could report initSubpoolData errors
- Checked that `filterSubpools` has no error-propagation mechanism — errors are logged and subpools are dropped
- The specific config trigger (BP + conditional job collision) is a hypothesis — the symptom (silent drop, no status) is confirmed

## Next steps

- Confirm the hypothesis: reproduce with the exact config from the maintainer's comment and verify the Validate error is the one firing
- Fix A (higher priority): deduplicate contexts in `GetTideContextPolicy` after merging BP and job sources, before calling `Validate()` — keep `Validate()` strict so `checkconfig` still catches explicit misconfigurations
- Fix B (hardening): propagate `initSubpoolData` errors to the tide status context as `github.StatusError` — prevents any future init error from being silent
- Label with `kind/bug`, `area/tide`, `help-wanted`

## Open questions

- Should `checkconfig` be updated to detect this configuration pattern and warn proactively?
- Is "required wins over required-if-present" always the right resolution, or are there cases where conditional semantics should take precedence?
