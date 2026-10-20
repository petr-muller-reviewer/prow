---
issue: kubernetes-sigs/prow#777
title: "tide: context checker error silently drops PR instead of reporting via status context"
state: open
labels: ""
main_sha: 444074c659e1d2626895fbc542f7cc6ccfb19e16
triaged_at: 2026-10-20T23:36:58Z
verdict: accepted
refresh_log:
  - since: 2026-07-01T22:19:04Z
    summary: Confirmed the BP/job context collision; checkconfig validation and config fixes landed, and Tide status reporting is in open PR #938.
advice:
  advised_at: 2026-10-20T23:38:35Z
  based_on_triaged_at: 2026-10-20T23:36:58Z
---

## What the issue reports

- A context-policy validation error can cause Tide to drop an entire subpool without setting a useful Tide status on the affected PR.
- The suspected collision is an explicit branch-protection required context that is also a conditionally triggered Prow job context.
- Since previous triage:
  - The collision mechanism was confirmed in a follow-up comment by petr-muller on 2026-07-01; maintainer Prucek asked whether Tide should also report the error, and petr-muller agreed that stronger Tide reporting would help.
  - PR #791 merged on 2026-07-02 and made `checkconfig` detect these BP/job collisions. OpenShift config PR #81365 also merged that day to remove collisions in seven repos; a later revert PR #81371 was closed without merging.
  - PR #938, opened 2026-09-14, remains open and proposes reporting excluded PRs with an error on their Tide status context. It explicitly fixes #777 and also addresses related issue #937.

## Findings

### [cause] Context list collision in GetTideContextPolicy (confirmed)
- detail: With `from-branch-protection: true`, an explicitly configured BP required context enters `required`, while a matching conditionally triggered, non-optional presubmit enters `requiredIfPresent`. `Validate()` rejects the overlap. The follow-up comment by petr-muller on 2026-07-01 confirmed this exact path using `ci/prow/test` and a `skip_if_only_changed` presubmit.
- evidence: `pkg/config/tide.go:962-987`, `pkg/config/branch_protection.go:604-638`, issue comment 2026-07-01T22:32:46Z

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

### [related-pr] checkconfig now detects branch-protection context collisions
- ref: kubernetes-sigs/prow#791
- relevance: Merged 2026-07-02; adds literal branches from branch-protection config to `validateTideContextPolicy` and tests detection of the BP-required/conditionally-triggered collision. This catches the configuration problem before runtime but does not add a Tide status for a PR when runtime initialization fails.

### [related-pr] Tide reports excluded PRs on their status context
- ref: kubernetes-sigs/prow#938
- relevance: Open; handles per-PR `initSubpoolData` failures as exclusions and reports `StatusError` with the merge-requirements error before building the context checker. Its approach matches the existing hardening recommendation and the maintainer's preference for surfacing the error.

### [related-pr] OpenShift config collision fixes
- ref: openshift/release#81365
- relevance: Merged 2026-07-02; removes colliding branch-protection contexts in seven affected repos, after PR #791 made `checkconfig` detect them. PR openshift/release#81371 proposed reverting some branch-protection changes but was closed without merging.

### [related-issue] Tide status missing for PRs excluded after changed-file fetch errors
- ref: kubernetes-sigs/prow#937
- relevance: Related symptom from a different exclusion path; open PR #938 addresses it alongside this issue.

## Checked

- Confirmed the error message in the issue matches `pkg/config/tide.go:869`
- Confirmed `checkconfig` validates context policies at config-load time but cannot catch runtime collisions from `from-branch-protection: true`
- Verified the status controller already uses `github.StatusError` for merge conflicts and blockers — same pattern could report initSubpoolData errors
- Checked that `filterSubpools` has no error-propagation mechanism — errors are logged and subpools are dropped
- The specific config trigger (BP required context + matching conditional job context) is confirmed by the 2026-07-01 issue comment and by the collision test added in PR #791
- PR #791 changes only `cmd/checkconfig`; it detects the config collision but does not surface runtime Tide status errors
- PR #938 changes `pkg/tide/status.go` and `pkg/tide/tide.go` to keep the affected PR's exclusion reason and report it as a Tide error status
- PR #81365 updates seven OpenShift repo configs; PR #81371, which proposed reverting some of those branch-protection changes, was closed unmerged

## Next steps

- Review and track open PR #938; its per-PR exclusion and `github.StatusError` reporting match the remaining user-visible problem and the maintainer's stated preference
- Decide whether runtime deduplication in `GetTideContextPolicy` is still desirable: PR #791 catches the config pattern and known OpenShift collisions were corrected, while Tide's runtime validation remains strict
- Label with `kind/bug`, `area/tide`, and `triage/accepted`; help wanted is unnecessary while PR #938 is active

## Advice

1. **Review and track PR #938; avoid duplicate implementation.** It is open and mergeable, explicitly fixes #777, and its status reporting addresses the remaining issue. PR #791 and openshift/release#81365 handle config validation and known config collisions, not Tide's missing status.

   ```sh
   gh pr view 938 --repo kubernetes-sigs/prow
   gh pr diff 938 --repo kubernetes-sigs/prow
   ```

2. **Apply the issue labels.** The issue is accepted but currently has no labels. These existing labels identify the bug, Tide area, and accepted status; `help wanted` is not needed while a fix is already in flight.

   ```sh
   gh issue edit 777 --repo kubernetes-sigs/prow --add-label "kind/bug,area/tide,triage/accepted"
   ```

## Open questions

- Should `checkconfig` be updated to detect this configuration pattern and warn proactively?
- Is "required wins over required-if-present" always the right resolution, or are there cases where conditional semantics should take precedence?
