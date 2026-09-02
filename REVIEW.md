---
pr: kubernetes-sigs/prow#735
title: "trigger: add /test-required command"
head_sha: 124f7da44fcbfd71622252c9d7459ecd1e267ee4
base: main
reviewed_at: 2026-08-08T19:41:04Z
verdict: needs-discussion
refresh_log:
  - from_sha: 11e9af6600623d7faec3c382e45e4c538ad66b7e
    to_sha: 11e9af6600623d7faec3c382e45e4c538ad66b7e
    at: 2026-06-01T15:50:53Z
    summary: "No code change. Maintainer Prucek posted inline review confirming NeedsExplicitTrigger() should be removed, and issue comment requesting help entry."
  - from_sha: 11e9af6600623d7faec3c382e45e4c538ad66b7e
    to_sha: 124f7da44fcbfd71622252c9d7459ecd1e267ee4
    at: 2026-06-15T12:44:06Z
    summary: "Author addressed both [should-fix] findings: removed NeedsExplicitTrigger check, added help provider entry and plugin description."
  - from_sha: 124f7da44fcbfd71622252c9d7459ecd1e267ee4
    to_sha: 124f7da44fcbfd71622252c9d7459ecd1e267ee4
    at: 2026-06-30T10:30:15Z
    summary: "No code change. Prucek /lgtm'd on 2026-06-17. Author cc'd stmcginnis and petr-muller for approval. PR awaiting approved label."
  - from_sha: 124f7da44fcbfd71622252c9d7459ecd1e267ee4
    to_sha: 124f7da44fcbfd71622252c9d7459ecd1e267ee4
    at: 2026-08-08T17:20:19Z
    summary: "No code change. petr-muller posted the remaining [should-fix]/[question] findings as an issue comment on 2026-06-30. Amulyam24 replied on 2026-08-04 agreeing to remove the context-check inconsistency and make NeedsExplicitTrigger handling respect require_manually_triggered_jobs, but has not yet pushed the change."
  - from_sha: 124f7da44fcbfd71622252c9d7459ecd1e267ee4
    to_sha: 124f7da44fcbfd71622252c9d7459ecd1e267ee4
    at: 2026-08-08T19:41:04Z
    summary: "No code change. Deep design pass: verified the #729 requester's actual config, traced Tide's treatment of manually-triggered required jobs, and converged on a new filter definition. petr-muller posted a design comment proposing /test-manual-required. Prior [should-fix] findings on context-checking and require_manually_triggered_jobs are superseded by a single predicate change."
---

## Summary

Adds `/test-required`: triggers non-optional presubmits whose context has not been reported yet, including explicitly-triggered jobs (`forcedToRun = true`). New `TestRequiredFilter` in `pkg/pjutil/filter.go`, wired into `commentMatchesTrigger` and `PresubmitFilter`. +161/-0 across 4 files. Implementation is clean; the disagreement is about which set of jobs the command should select.

Design converged this session: the command should select `ContextRequired() && NeedsExplicitTrigger()` unconditionally — the exact complement of `/test all` — with no context inspection and no branch-protection/Tide config awareness. Requires rename. Net diff is smaller than what is on the branch.

## Findings

### Resolved

#### [should-fix] NeedsExplicitTrigger check incorrectly excluded explicitly-triggered required jobs
- **Addressed in 124f7da44**: filter now returns `!trf.allContexts.Has(ps.Context), ps.NeedsExplicitTrigger(), false`.

#### [should-fix] Missing help provider entry for /test-required
- **Addressed in 124f7da44**: `pluginHelp.AddCommand` entry added.

#### [nit] Plugin long description omits /test-required
- **Addressed in 124f7da44**.

### Superseded

#### [should-fix] Context-checking is inconsistent with the /test command family
#### [should-fix] Command assumes NeedsExplicitTrigger jobs are required, but Tide defaults disagree
#### [question] Should the command respect `require_manually_triggered_jobs`?
- Both should-fixes and the question are **superseded by the single predicate change below**. Answer to the question is now a firm **no**: the trigger plugin should not read branch-protection or Tide config. See `[should-fix] Filter predicate`.

### Remaining

### [should-fix] Filter predicate should be `ContextRequired() && NeedsExplicitTrigger()`, forced unconditionally
- where: `pkg/pjutil/filter.go:269-287`, `pkg/pjutil/filter.go:317-323`
- concern: current predicate is a `/retest`-shaped context check under a `/test` name, and its `NeedsExplicitTrigger()` handling only feeds `forced`. Moving that term into the match predicate makes the command the exact complement of `TestAllFilter` (`!p.NeedsExplicitTrigger()`, `filter.go:150`): two disjoint sets whose union is every non-optional job. The set is then small by construction, so unconditional triggering costs nothing and the context check becomes unnecessary rather than merely inconsistent. No config plumbing needed, so the `require_manually_triggered_jobs` question disappears entirely.
- excerpt: |
    // current
    func (trf *TestRequiredFilter) ShouldRun(ps config.Presubmit) (bool, bool, bool) {
        if ps.Optional {
            return false, false, false
        }
        return !trf.allContexts.Has(ps.Context), ps.NeedsExplicitTrigger(), false
    }

    // target
    func (trf *TestRequiredFilter) ShouldRun(ps config.Presubmit) (bool, bool, bool) {
        return ps.ContextRequired() && ps.NeedsExplicitTrigger(), true, false
    }
- note: NOT merely "drop the context check" — dropping it alone yields "trigger every non-optional job unconditionally", including `always_run: true` jobs that already ran. That is the variant the author described implementing on 2026-08-04 and it is the wasteful one.
- side effects: `allContexts` field and constructor arg are deleted; the `contextGetter()` call in `PresubmitFilter` (`filter.go:317-323`) goes away, so the command stops issuing a `GetCombinedStatus` call and the block collapses to match the `NewTestAllFilter()` line below it.
- works for #729 unchanged: the requester's jobs are `always_run: false, optional: false` with no conditional matcher, so they satisfy the predicate with no config change on their side.

### [should-fix] Command name is misleading
- where: `pkg/pjutil/filter.go:39`, help provider entry in `pkg/plugins/trigger/trigger.go`
- concern: `/test-required` implies all required jobs, including `always_run: true` ones, which the target predicate deliberately excludes. Proposed `/test-manual-required`. If the name stays `/test-required`, the exclusion needs prominent documentation in the help entry.
- excerpt: |
    var TestRequiredRe = regexp.MustCompile(`(?m)^/test-required\s*$`)

### [nit] Use `ContextRequired()` rather than `!ps.Optional`
- where: `pkg/pjutil/filter.go:281`
- concern: `ContextRequired()` is `!ps.Optional && !ps.SkipReport` (`pkg/config/jobs.go:550-552`). A `skip_report: true` job posts no context and cannot be required in any meaningful sense. `RetestRequiredFilter` (`filter.go:254`) uses bare `ps.Optional`, so precedent is mixed.

### [nit] Unit and integration tests need inverting, not extending
- where: `pkg/pjutil/filter_test.go`, `pkg/plugins/trigger/generic-comment_test.go:595-649`
- concern: under the target predicate, the "skips jobs with already-reported contexts" expectations invert, and a new case should assert that `always_run: true` required jobs are **not** triggered. The previously-noted missing coverage for already-reported contexts becomes moot.

### [nit] No job provenance label
- where: `pkg/plugins/trigger/generic-comment.go:164-167`
- concern: `/retest` and `/retest-required` add `kube.RetestLabel`. This command does not. Semantically correct, minor observability gap.

### [question] Dash-command form vs `/test <arg>` form
- where: `pkg/pjutil/filter.go:30,39`
- concern: `TestAllRe` is `^/test all,?($|\s.*)` — a set selector expressed as an argument. Dashes are used for command variants (`/retest-required`, `/ok-to-test`). A set selector under `/test` arguably belongs in argument form (`/test manual-required`). Not blocking; raised in the posted comment for the author/maintainers to settle.

## Checked

Verified in the requester's environment (`metal3-io/project-infra`):
- `prow/config/config.yaml`: `tide.context_options.from-branch-protection: true`; `branch-protection:` is only `protect: true` for orgs `Nordix` and `metal3-io`. **No `require_manually_triggered_jobs` anywhere.**
- Target jobs (`cluster-api-provider-metal3.yaml:170-185`, `ip-address-manager.yaml:149-164`, `ironic-image.yaml:52-67`): `always_run: false, optional: false`, no conditional matcher → `NeedsExplicitTrigger() && ContextRequired()`.
- `baremetal-operator.yaml:124-205` carries ~10 `always_run: false, optional: true` e2e jobs — the concrete reason `/test all` is unacceptable to them.
- No branchprotector deployment found in the repo (only a doc-comment mention). The GitHub `Required` badge in the #729 screenshot is therefore likely hand-managed outside Prow, since `BranchRequirements` would place these contexts in `requiredIfPresent`, and branchprotector replaces rather than unions the context list (`cmd/branchprotector/protect.go:542`).

Existing commands do not cover the use case:
- `/retest-required` → `RetestRequiredFilter` (`filter.go:254`) → `RetestFilter` (`filter.go:236`): `failed || (!p.NeedsExplicitTrigger() && !allContexts.Has(p.Context))`. Never-run manual job: `failed` false, second clause short-circuited → not triggered.
- `/test all` → `TestAllFilter` (`filter.go:150`) excludes `NeedsExplicitTrigger()` by construction.

Config-level workarounds that exist today:
- Shared `trigger:` regex across jobs is legal — only job *names* are checked for uniqueness (`config.go:2306-2336`); `CommandFilter` (`filter.go:127`) returns `forced = true`, so matched jobs run despite `always_run: false`.
- `rerun_command` is mandatory when `trigger` is set (`config.go:3134`; defaulting only fires when both are empty, `config.go:3245`) and must itself match the trigger regex (`config.go:3274-3275`). It is copied into the ProwJob (`pjutil.go:177`) and printed in the failure report (`report/report.go:313`), so a shared `rerun_command` makes a single failure advertise a rerun of the whole set. Widen `trigger`, keep `rerun_command` per-job.
- `run_before_merge: true` (`jobs.go:216-220`) is the per-job equivalent of `require_manually_triggered_jobs` — same `forceRun` expression, but does not feed `BranchRequirements`, so it does not change what GitHub requires.

Tide's treatment of manually-triggered required jobs (without the flag) — enforced but unmanaged:
- `jobIsRequiredByTide` (`tide/github.go:443-445`) = `ContextRequired() || RunBeforeMerge` → passes.
- `forceRun` (`tide.go:1708`) false → `ps.ShouldRun(branch, changes, false, false)` returns false (`jobs.go:518-530` with `RegexpChangeMatcher.ShouldRun` returning `determined=false`, `jobs.go:449-456`) → job absent from `sp.presubmits[pr.Number]`.
- Consequence: `accumulate` (`tide.go:1108-1122`) never sees it → never in `missingTests` → Tide never triggers or retests it.
- But `IsOptional` is false for `RequiredIfPresentContexts` (`config/tide.go:1015-1016`), so `unsuccessfulContexts` (`tide.go:865-877`) still gates on it: failure → PR filtered from pool (`tide.go:781`) with no possible retrigger; pending → filtered as "not Prow-controlled" (`tide.go:785-788`); success → accepted forever, never revalidated when the base moves (the base-SHA staleness check in `accumulate` only covers jobs in the required set).
- With `require_manually_triggered_jobs` + `from-branch-protection` (`tide.go:2395-2405`): `forceRun` true → Tide triggers the job itself once the PR is in the pool. Introduced by `c13d06d6c` (2024-01-21, "tide: Fix require_manually_triggered_jobs tide handling", *"resulted in missing protection for this kind of jobs"*). The auto-trigger machinery itself is old (`75b0d3a36`, 2019-08-03).
- Batches already use union semantics: `presubmitsForBatch` (`tide.go:1731-1776`) recomputes from `batchChanges(prs)` (union of changed files) and is the single chokepoint for both `pickBatch` (`:1242`) and `accumulateBatch` (`:1005`), so pick-time and validate-time cannot drift.

Unchanged from earlier reviews:
- Filter ordering (test-required after retest-required, before test-all) correct in `AggregateFilter`.
- `commentMatchesTrigger` wiring correct; help pruning works (`/test-required` matches `anyTestRe`, `help.go:33`).
- Purely additive; no config/API/behavioral change to existing commands; cleanly revertible; no security concerns.

## Open questions
- Naming: `/test-manual-required` (precise, long) vs `/test-required` + prominent docs vs argument form `/test manual-required` matching the `/test all` precedent.
- Does metal3-io hand-manage GitHub branch protection for these contexts? Their answer determines whether `require_manually_triggered_jobs: true` or `run_before_merge: true` is the config fix they actually want, independent of this PR.
- Follow-up issue candidate against Tide, independent of this PR: make a *present* context imply *managed*, i.e. add "context already present on this PR" as a `forceRun` disjunct in `presubmitsByPull` (`tide.go:1708`) so a maintainer's manual trigger produces a genuinely first-class required check (retriggered on base drift, self-healing on failure) without the org-wide flag. Same defect class as `c13d06d6c`, adjacent axis. Open risks: batch composition (union over per-PR context presence), and `headContexts` fetch cost (possibly already cached — see `tide.go:1211-1212`).

## Activity since last review (2026-08-08T17:20:19Z)

- **2026-08-08** — petr-muller posted a design comment proposing `/test-manual-required` with the predicate above, explicitly rejecting `require_manually_triggered_jobs` awareness ("I dislike trying to be too smart"), and noting `/test-required` would need prominent documentation if kept.
- **Correction still owed on that comment**: its second bullet claims that with `require_manually_triggered_jobs` set, Tide will not merge "until someone triggers them manually and they pass". Tide triggers them itself (`forceRun`, `tide.go:1708`). The command's value in that configuration is *earliness* — starting jobs before the PR enters the pool (i.e. before lgtm/approved) — not necessity.
- Author has not yet pushed the version described on 2026-08-04 (unconditional trigger + `require_manually_triggered_jobs` gating). Steering away from it is time-sensitive.
