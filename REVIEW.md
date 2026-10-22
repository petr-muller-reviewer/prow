---
pr: kubernetes-sigs/prow#908
title: "`override`: abort in-flight jobs so they cannot clobber `/override` or `/override-sticky`"
head_sha: a3936e14f4cde303f9d6faea039c6345097a3c2b
base: main
reviewed_at: 2026-08-31T16:04:33Z
verdict: request-changes
---

## What this PR does

- Adds `abortActiveJob` (and helper `prowJobSelectorForPR`) in `pkg/plugins/override/override.go`, which lists ProwJobs for a PR by org/repo/pull/type label selector, filters by job name, and sets `AbortedState` on any that aren't yet complete.
- Wires `abortActiveJob` into the commit-status override loop (`handleOverride`, called right after the override's synthetic success `ProwJob` is created for a context) so a still-running real job for that context can't later have crier report its actual result over the override.
- Leaves `AbortedState` set but does not call `SetComplete`, deliberately, so Plank/Jenkins still owns tearing down the running pod/build and completing the job — comment at override.go:186-190 documents this.
- No changes to the Checks-API (App-auth) branch of `handleOverride`, which creates `CheckRun`s for `checkrunContexts` independently of the `statuses`/`abortActiveJob` loop.

## Findings

### [blocking] Checks-API override path never aborts in-flight jobs
- where: `pkg/plugins/override/override.go:632-658`
- concern: The new abort protection is only invoked from the commit-status loop (`abortActiveJob` call at line 623, inside the loop iterating `statuses`). The separate `if oc.UsesAppAuth()` block that follows (632-658) marks checkrun contexts as successful via `oc.CreateCheckRun` but never creates a ProwJob or calls `abortActiveJob` for them. For a repo where `oc.UsesAppAuth()` is true, `/override`ing a checkrun-backed context whose presubmit is still running leaves that ProwJob running; when it completes, crier reports its real result via the Checks API and overwrites the override's success checkrun — the exact clobbering bug this PR's commit message says it fixes, just left unfixed for App-auth repos.
- excerpt: |
    if oc.UsesAppAuth() {
    	for _, checkrun := range checkrunContexts {
    		if overrides.Has(checkrun.Context) {
    			prowOverrideCR := github.CheckRun{
    				Name:       checkrun.Context,
    				...
    			}
    			if _, err := oc.CreateCheckRun(org, repo, prowOverrideCR); err != nil {
    				...
    			}
    			done.Insert(checkrun.Context)
    		}
    	}
    }

### [should-fix] Duplicates existing job-abort helper from the trigger package
- where: `pkg/plugins/override/override.go:174-186`
- concern: `prowJobSelectorForPR` + `abortActiveJob` reimplement the same org/repo/pull/type selector and List+Complete()-check+Update+IsConflict-swallow pattern as `labelSelectorForPR` + `abortAllJobs` in `pkg/plugins/trigger/pull-request.go`. Two near-identical implementations now have to be kept in sync; a future change to selector construction or conflict handling (e.g. adding a job-name label instead of client-side filtering) risks drifting between the two packages.
- excerpt: |
    func prowJobSelectorForPR(org, repo string, number int) (klabels.Selector, error) {
    	set := klabels.Set{
    		kube.OrgLabel:         org,
    		kube.RepoLabel:        repo,
    		kube.PullLabel:        strconv.Itoa(number),
    		kube.ProwJobTypeLabel: string(prowapi.PresubmitJob),
    	}
    	...
    }

### [nit] Redundant List() per overridden context
- where: `pkg/plugins/override/override.go:623` (call site inside the `for _, status := range statuses` loop)
- concern: `abortActiveJob` issues a fresh `List()` call (plus one `Update()` per matching job) on every loop iteration, i.e. once per overridden context, instead of listing the PR's presubmit ProwJobs once per invocation and filtering by job name in memory. Overriding several contexts on the same PR multiplies API calls proportionally to the number of contexts for no functional benefit.
- excerpt: |
    contextsWithCreatedJobs.Insert(status.Context)
    abortActiveJob(oc, log, org, repo, number, pre.Name)

## Checked

- `abortActiveJob` correctly skips completed jobs (`pj.Complete()` check) and swallows `IsConflict` errors from concurrent updates without failing the override.
- The comment at override.go:183-190 explaining why `SetComplete` is deliberately not called (leaves teardown to Plank/Jenkins) is accurate against the implementation.
- The commit-status loop's placement of `abortActiveJob` (only after a new override ProwJob was successfully created, via `contextsWithCreatedJobs`) avoids aborting jobs when no override job exists yet.

## Open questions

- Is the omission of `abortActiveJob` from the `UsesAppAuth()`/checkrun branch intentional (e.g. Checks-API repos handle in-flight jobs differently elsewhere) or an oversight? If checkrun-backed repos are in scope for this fix, that branch needs the same treatment.
- Given the duplication with `pkg/plugins/trigger/pull-request.go`'s `abortAllJobs`/`labelSelectorForPR`, would extracting a shared helper be worth doing now rather than after a second consumer appears?

## Followups

Accepted: 4 (2 deliver, 2 probe; retained from the previous walkthrough). Skipped: 1.

Context: PR #908 merged on 2026-08-31 at `5765df26189ddc41cb9c975e89d613f6c26282ef`. The override files at that merge match reviewed head `a3936e14f4cde303f9d6faea039c6345097a3c2b`. No deferred implementation requests were found in the PR discussion. Followup agents must work on the current default branch and check whether later changes have already addressed each task.

Validation note: the saved Checks-API blocker above is not substantiated by the inspected reporting path. `pkg/github/report/report.go:86` posts commit statuses, not check runs, and `pkg/tide/tide.go:2266` prefers successful duplicate check runs. The branch's missing abort call is real, but the asserted crier check-run overwrite is not established. Followup 1 validates this before proposing a behavior change; the historical finding and frontmatter are preserved.

Revisited at 2026-10-22T16:14:28Z: the four accepted handoffs below remain applicable to the inspected PR head. No deferred implementation requests or added TODOs were found in the PR discussion/diff. Skipped the additional documentation candidate: explain active Prow job abortion, reporting suppression, and cancellation without automatic restart in `site/content/en/docs/jobs.md` and override command help. This inspection does not establish whether later changes on the default branch have addressed the handoffs; each prompt retains that prerequisite.

### 1. Validate the saved Checks-API finding

- Category: tests
- Necessity: should — establish whether a real lifecycle gap exists before changing behavior.
- Mode: probe
- Where: `pkg/plugins/override/override.go:638-658`, `pkg/github/report/report.go`, `pkg/tide/tide.go`.

```text
In kubernetes-sigs/prow, following PR #908 — "`override`: abort in-flight jobs so they cannot clobber `/override` or `/override-sticky`" (merge 5765df26189ddc41cb9c975e89d613f6c26282ef), work on the current default branch and first check whether subsequent changes already addressed this task.

Mode: probe. Write temporary tests exercising check-run-only overrides, matching Prow presubmits, required-context fallback, and subsequent job completion for both /override and /override-sticky. Trace crier's actual reporting API and Tide's duplicate-check selection. Establish whether an active ProwJob can invalidate an accepted override. Do not assume UsesAppAuth changes crier to Checks API reporting: at the merge, crier posts commit statuses and Tide prefers successful duplicate check runs. The saved REVIEW.md blocker is a hypothesis to validate, not an established bug.

Acceptance criteria: exercise each named behavior and classify every failure with evidence, distinguishing implementation bugs from incorrect expectations or test mistakes. Report evidence confirming or correcting the saved finding. Apply only minimal fixes for demonstrated bugs. Remove all exploratory tests before finishing; if no bug exists, leave no files changed. Report each failing scenario so the maintainer can choose whether to retain regression coverage separately.

Scope: exclude external check-run cancellation, unrelated override semantics, and cosmetic cleanup. Do not publish changes without an explicit user request.
```

### 2. Share the PR selector and job-abort mechanics

- Category: tech-debt
- Necessity: could — reduce maintenance drift; duplication alone is not a demonstrated failure.
- Mode: deliver
- Where: `pkg/plugins/override/override.go:175-217`, `pkg/plugins/trigger/pull-request.go:171-219`.

```text
In kubernetes-sigs/prow, following PR #908 — "`override`: abort in-flight jobs so they cannot clobber `/override` or `/override-sticky`" (merge 5765df26189ddc41cb9c975e89d613f6c26282ef), work on the current default branch and first check whether subsequent changes already addressed this task.

Mode: deliver. Extract the shared PR presubmit selector and narrowly reusable abort mechanics from pkg/plugins/override/override.go and pkg/plugins/trigger/pull-request.go into an appropriate shared package, such as pkg/pjutil. Preserve caller policies: override filters by job name and sets Report=false; trigger handles all incomplete PR presubmits without changing Report. Neither should set CompletionTime; controllers retain workload teardown ownership. Preserve existing error/conflict behavior in this cleanup.

Acceptance criteria: both callers use the shared mechanics, their distinct selection/reporting/error behavior is preserved, and focused tests for override and trigger pass. Coordinate with any separately accepted multi-context batching change; preserve that behavior if already implemented.

Scope: avoid a generic job-management framework, unrelated behavior changes, or speculative conflict-handling fixes. Do not publish changes without an explicit user request.
```

### 3. List PR jobs once for a multi-context override

- Category: cleanup
- Necessity: could — reduce API calls; no current bottleneck was demonstrated.
- Mode: deliver
- Where: `pkg/plugins/override/override.go:598-623`.

```text
In kubernetes-sigs/prow, following PR #908 — "`override`: abort in-flight jobs so they cannot clobber `/override` or `/override-sticky`" (merge 5765df26189ddc41cb9c975e89d613f6c26282ef), work on the current default branch and first check whether subsequent changes already addressed this task.

Mode: deliver. Refactor pkg/plugins/override/override.go so a multi-context command lists the PR's presubmit ProwJobs once and reuses that result when aborting selected jobs. Preserve partial-success behavior: abort a job only after its synthetic override ProwJob was successfully created, retain processing already completed if a later creation fails, and publish each success status after its abort attempt. Preserve Report=false, AbortedState, controller-owned completion, and isolation from unrelated jobs and PRs.

Acceptance criteria: focused coverage asserts one List call for multiple overridden contexts, correct job-name/context mapping, and correct partial progress when a later synthetic-job creation fails. Verify both /override and /override-sticky continue to work. Run focused override tests. Coordinate with any accepted shared-helper refactor and retain its caller-policy distinctions.

Scope: exclude caching across commands, conflict-handling changes, and unrelated override semantics. Do not publish changes without an explicit user request.
```

### 4. Exercise abort conflicts and controller races

- Category: reliability
- Necessity: should — determine which races can still allow a real job to report over an override.
- Mode: probe
- Where: `pkg/plugins/override/override.go:201-215`, adjacent controller/reporting paths.

```text
In kubernetes-sigs/prow, following PR #908 — "`override`: abort in-flight jobs so they cannot clobber `/override` or `/override-sticky`" (merge 5765df26189ddc41cb9c975e89d613f6c26282ef), work on the current default branch and first check whether subsequent changes already addressed this task.

Mode: probe. Write temporary tests exercising deterministic interleavings in the override abort path: an unrelated resource-version update between List and Update; job completion before the abort update; List/Update failures; and a reporter holding an older job snapshot while the override is published. Cover both /override and /override-sticky. At the merge, Update conflicts are silently dropped and other abort API failures only logged while the handler can still report override success.

Acceptance criteria: exercise every named scenario and classify failures with evidence, distinguishing real API/reporting behavior from fake-client artifacts or incorrect test expectations. For demonstrated bugs, apply the smallest fix, using bounded re-read/retry where appropriate while preserving controller-owned completion and workload teardown. Never retry by blindly overwriting newer job state. Report residual races that cannot be fixed locally and each failing scenario so the maintainer can choose whether to retain regression coverage separately. Remove all exploratory tests before finishing; if no bug exists, leave no files changed.

Scope: exclude broad controller/reporting redesign, unrelated trigger behavior, and cosmetic cleanup. Preserve any already-landed shared-helper or batching refactor. Do not publish changes without an explicit user request.
```
