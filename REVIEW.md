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
