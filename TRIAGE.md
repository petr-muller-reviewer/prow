---
issue: kubernetes-sigs/prow#789
title: "Override plugin: follow-up cleanup from sticky override work"
state: open
labels:
main_sha: d3dc1c7b61b3f489e1f2d0f65b62b18dc6d8d079
triaged_at: 2026-10-10T16:35:47Z
refresh_log:
  - at: 2026-10-10T16:35:47Z
    since: 2026-07-27T00:34:55Z
    summary: "Issue remains open and unlabeled; open PR #808 addresses the BaseSHA reuse item while synthetic ProwJob cleanup remains outstanding."
advice:
  advised_at: 2026-10-10T16:38:09Z
  based_on_triaged_at: 2026-10-10T16:35:47Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [help-wanted, kind/cleanup, area/plugins]
---

## What the issue reports

- The synthetic successful ProwJob created by `/override` may be redundant now that Tide can read the embedded BaseSHA from the status description.
- `/override` should reuse a BaseSHA already embedded in the original status, falling back to the current base ref only when none is present.

**Since previous triage:**

- No comments were added after `2026-07-27T00:34:55Z`; the issue remains open with no labels.
- On 2026-07-27, SaaiAravindhRaja cross-referenced `kubernetes-sigs/prow#808`, an open PR for the BaseSHA reuse item. It adds per-status embedded SHA reuse with a cached fallback and tests; its description explicitly keeps synthetic ProwJob creation.
- The timeline also references `SaaiAravindhRaja/prow#1`, an open fork PR with the same title and override changes as #808, plus dependency-file changes.

## Findings

### [cause] synthetic ProwJob creation is redundant since PR #778
- detail: PR #778 added `prowJobsFromContexts`, which reconstructs an in-memory (non-persisted) passing `ProwJob` for Tide's individual-PR merge decision straight from the GitHub status description's embedded BaseSHA. This makes the real cluster ProwJob that `/override` creates unnecessary for Tide's own bookkeeping — Tide no longer needs the object to survive Sinker reaping to avoid a spurious retest.
- evidence: `pkg/tide/tide.go:1110-1145`

### [cause] baseSHA is always fetched fresh, never reused from an existing embedded value
- detail: `/override` unconditionally calls `baseSHAGetter()` to get the current base-branch tip, even when the status being overridden already has a BaseSHA embedded in its description from the original job run (via `config.ContextDescriptionWithBaseSha`). This can override the wrong BaseSHA versus the one the failing/pending job actually ran against.
- evidence: `pkg/plugins/override/override.go:626`

### [related-code] synthetic ProwJob creation (candidate for removal)
- where: `pkg/plugins/override/override.go:651-670`
- excerpt: |
    if pre != nil {
        pj := pjutil.NewPresubmit(*pr, baseSHA, *pre, e.GUID, nil)
        now := metav1.Now()
        pj.Status = prowapi.ProwJobStatus{
            StartTime:      now,
            CompletionTime: &now,
            State:          prowapi.SuccessState,
            Description:    descFn(user),
            URL:            e.HTMLURL,
        }
        log.WithFields(pjutil.ProwJobFields(&pj)).Info("Creating a new prowjob.")
        if _, err := oc.Create(context.TODO(), &pj, metav1.CreateOptions{}); err != nil {
            resp := fmt.Sprintf("Failed to create override job for %s", status.Context)
            log.WithError(err).Warn(resp)
            return oc.CreateComment(org, repo, number, plugins.FormatResponseRaw(e.Body, e.HTMLURL, user, resp))
        }
        contextsWithCreatedJobs.Insert(status.Context)
    }

### [related-code] baseSHA fetched once per invocation, unconditionally
- where: `pkg/plugins/override/override.go:626`
- excerpt: |
    baseSHA, err := baseSHAGetter()
    if err != nil {
        resp := "Cannot get base ref of PR"

### [related-code] BaseSHA already embedded in status description on every override
- where: `pkg/plugins/override/override.go:673`
- excerpt: |
    status.Description = config.ContextDescriptionWithBaseSha(descFn(user), baseSHA)

### [related-code] Tide reconstructs a synthetic in-memory ProwJob from the status description
- where: `pkg/tide/tide.go:1110-1128`
- excerpt: |
    for _, headContext := range headContexts {
        if headContext.State != githubql.StatusStateSuccess {
            continue
        }
        desc := string(headContext.Description)
        baseSHAForContext := config.BaseSHAFromContextDescription(desc)
        if config.IsSkipRetest(desc) || (baseSHAForContext != "" && baseSHAForContext == baseSHA) {
            passingCurrentContexts = append(passingCurrentContexts, string(headContext.Context))
        }
    }

### [related-code] Sinker reaps ProwJobs after max_prowjob_age
- where: `cmd/sinker/main.go:360-376`
- excerpt: |
    isFinished.Insert(prowJob.ObjectMeta.Name)
    if time.Since(prowJob.Status.StartTime.Time) <= maxProwJobAge {
        continue
    }
    if err := c.prowJobClient.Delete(c.ctx, &prowJob); err == nil {

### [related-code] BaseSHA embed/extract helpers already exist
- where: `pkg/config/config.go:3424-3459`
- excerpt: |
    func ContextDescriptionWithBaseSha(humanReadable, baseSHA string) string { ... }
    func BaseSHAFromContextDescription(description string) string { ... }

### [related-code] Tide's only live ProwJob lookup is gated on Pending state
- where: `pkg/tide/tide.go:1323-1358`
- excerpt: |
    if headContext.State == githubql.StatusStatePending {
        ... c.prowJobClient.List(...) // isRetestEligible
    }

### [related-pr] originating PR for this follow-up
- ref: kubernetes-sigs/prow#778
- relevance: "sticky-override-for-head", merged as 0633879af; introduced BaseSHA embedding in status descriptions and `prowJobsFromContexts`. Both items in #789 were split out of its review thread.

### [related-pr] BaseSHA reuse follow-up in progress
- ref: kubernetes-sigs/prow#808
- relevance: Open PR "override: preserve status base SHA when overriding jobs" updates `baseSHAForStatus()` to prefer the status's embedded BaseSHA and fall back to a cached base-ref lookup. It addresses the second issue item and adds tests, while retaining synthetic ProwJob creation.

## Checked

- Re-anchored the relevant findings to `main_sha`: BaseSHA is embedded via `config.ContextDescriptionWithBaseSha` (`override.go:673`); the synthetic ProwJob creation (`override.go:651-670`) and unconditional `baseSHAGetter()` call (`override.go:626`) remain.
- Since the earlier main SHA, `override.go` added active-job cancellation. `abortActiveJob()` runs after synthetic ProwJob creation (`override.go:670`) so removing the synthetic job must preserve that call to prevent an in-flight job from overwriting the override status.
- Verified Tide's `accumulate()` does not require a real cluster ProwJob for overridden contexts — `prowJobsFromContexts` reconstructs an equivalent in-memory job from the status description alone (`tide.go:1110-1145`).
- Verified Tide's only live ProwJob lookup (`isRetestEligible`) is gated on `Pending` state, so it never fires for overridden (`Success`) contexts — rules out a hidden dependency on the real ProwJob for retest eligibility.
- Searched for other consumers of override-created ProwJobs (metrics, spyglass, statusreconciler, crier) — none found; only Deck's live job-listing UI (`cmd/deck/main.go`) displays them until Sinker reaps them.
- Current `pkg/plugins/override/override_test.go` fixtures still do not exercise an input status with a pre-existing embedded BaseSHA differing from the fetched one. Open PR #808 adds that case and a fallback-caching case; its coverage is not in main yet.

## Next steps

- Apply labels: `/area plugins`, `/kind cleanup`, `/help-wanted`.
- Review/track open PR #808 for item 2. Its per-status lookup and cached fallback match the triage findings and it adds coverage for an embedded BaseSHA and the fallback path.
- Item 1 remains open for a separate change: remove synthetic ProwJob creation in `handle()` while preserving the `abortActiveJob()` side effect.
- For item 1, call out that overridden contexts will stop appearing as a distinct job row in Deck's job list (GitHub status/check itself is unaffected) — this is an observable UX change, not just internal cleanup.

## Advice

- **Review the existing BaseSHA change; don't duplicate it.** The issue is still open, and canonical PR #808 is open with no review decision recorded. It covers reuse of the original status BaseSHA but deliberately keeps synthetic ProwJob creation, so item 1 remains a separate follow-up. The fork copy, `SaaiAravindhRaja/prow#1`, is also open.
  ```sh
  gh pr diff 808 --repo kubernetes-sigs/prow
  ```
- **Apply the issue's triage and discovery labels.** The issue has no labels; the repo has `triage/accepted`, `kind/cleanup`, `area/plugins`, and `help wanted`. No milestone is assigned, and the triage has no release target to attach.
  ```sh
  gh issue edit 789 --repo kubernetes-sigs/prow --add-label "triage/accepted,kind/cleanup,area/plugins,help wanted"
  ```

## Open questions

- Should checkrun-only overrides (app-auth path, `pkg/plugins/override/override.go:576-597`) also get baseSHA-reuse treatment, or is item 2 scoped only to the status-based path? Not addressed by the issue text.
- Does anyone rely on Deck's job list showing override actions as a distinct job row (e.g. for auditing "who overrode what")? Worth a quick check before merging item 1, even though no code dependency was found.
