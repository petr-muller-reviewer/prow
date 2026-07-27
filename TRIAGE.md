---
issue: kubernetes-sigs/prow#789
title: "Override plugin: follow-up cleanup from sticky override work"
state: open
labels:
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:34:55Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 2
recommended_labels: [help-wanted, kind/cleanup, area/plugins]
---

## Findings

### [cause] synthetic ProwJob creation is redundant since PR #778
- detail: PR #778 added `prowJobsFromContexts`, which reconstructs an in-memory (non-persisted) passing `ProwJob` for Tide's individual-PR merge decision straight from the GitHub status description's embedded BaseSHA. This makes the real cluster ProwJob that `/override` creates unnecessary for Tide's own bookkeeping — Tide no longer needs the object to survive Sinker reaping to avoid a spurious retest.
- evidence: `pkg/tide/tide.go:1049-1075`

### [cause] baseSHA is always fetched fresh, never reused from an existing embedded value
- detail: `/override` unconditionally calls `baseSHAGetter()` to get the current base-branch tip, even when the status being overridden already has a BaseSHA embedded in its description from the original job run (via `config.ContextDescriptionWithBaseSha`). This can override the wrong BaseSHA versus the one the failing/pending job actually ran against.
- evidence: `pkg/plugins/override/override.go:518`

### [related-code] synthetic ProwJob creation (candidate for removal)
- where: `pkg/plugins/override/override.go:543-561`
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
- where: `pkg/plugins/override/override.go:518`
- excerpt: |
    baseSHA, err := baseSHAGetter()
    if err != nil {
        resp := "Cannot get base ref of PR"

### [related-code] BaseSHA already embedded in status description on every override
- where: `pkg/plugins/override/override.go:564`
- excerpt: |
    status.Description = config.ContextDescriptionWithBaseSha(descFn(user), baseSHA)

### [related-code] Tide reconstructs a synthetic in-memory ProwJob from the status description
- where: `pkg/tide/tide.go:1049-1075`
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
- where: `cmd/sinker/main.go:353-369`
- excerpt: |
    isFinished.Insert(prowJob.ObjectMeta.Name)
    if time.Since(prowJob.Status.StartTime.Time) <= maxProwJobAge {
        continue
    }
    if err := c.prowJobClient.Delete(c.ctx, &prowJob); err == nil {

### [related-code] BaseSHA embed/extract helpers already exist
- where: `pkg/config/config.go:3416-3441`
- excerpt: |
    func ContextDescriptionWithBaseSha(humanReadable, baseSHA string) string { ... }
    func BaseSHAFromContextDescription(description string) string { ... }

### [related-code] Tide's only live ProwJob lookup is gated on Pending state
- where: `pkg/tide/tide.go:1274-1298`
- excerpt: |
    if headContext.State == githubql.StatusStatePending {
        ... c.prowJobClient.List(...) // isRetestEligible
    }

### [related-pr] originating PR for this follow-up
- ref: kubernetes-sigs/prow#778
- relevance: "sticky-override-for-head", merged as 0633879af; introduced BaseSHA embedding in status descriptions and `prowJobsFromContexts`. Both items in #789 were split out of its review thread.

## Checked

- Confirmed both issue claims against current code at main_sha: BaseSHA is embedded via `config.ContextDescriptionWithBaseSha` (override.go:564); synthetic ProwJob creation (override.go:543-561) and unconditional `baseSHAGetter()` call (override.go:518) are both still present.
- Verified Tide's `accumulate()` (individual-PR merge path, `pkg/tide/tide.go:1080-1132`) does not require a real cluster ProwJob for overridden contexts — `prowJobsFromContexts` reconstructs an equivalent in-memory job from the status description alone.
- Verified Tide's only live ProwJob lookup (`isRetestEligible`, `pkg/tide/tide.go:1258-1305`) is gated on `Pending` state, so it never fires for overridden (`Success`) contexts — rules out a hidden dependency on the real ProwJob for retest eligibility.
- Searched for other consumers of override-created ProwJobs (metrics, spyglass, statusreconciler, crier) — none found; only Deck's live job-listing UI (`cmd/deck/main.go`) displays them until Sinker reaps them.
- Reviewed `pkg/plugins/override/override_test.go` fixtures — no existing test exercises an input status with a pre-existing embedded BaseSHA differing from the fetched one; item 2 is a genuine test-coverage gap, not just an implementation gap. Job-creation assertions (`jobs:` field in `TestHandle` cases) only check presence/absence per context, so removing creation only requires pruning those, not rewriting deep assertions.

## Next steps

- Apply labels: `/area plugins`, `/kind cleanup`, `/help-wanted`.
- Land as two separate, independent PRs: (1) remove synthetic ProwJob creation in `handle()`, (2) reuse embedded baseSHA via `config.BaseSHAFromContextDescription` with fallback to `baseSHAGetter()`.
- For PR (1): call out in the description that overridden contexts will stop appearing as a distinct job row in Deck's job list (GitHub status/check itself is unaffected) — this is an observable UX change, not just internal cleanup.
- For PR (2): add new test fixtures covering an input status with an embedded BaseSHA that differs from the fake `GetRef` value, and ideally assert `GetRef`/`baseSHAGetter` is skipped when one is present. Note different contexts can carry different embedded BaseSHAs, so this needs a per-context (not per-call) baseSHA, a bit more structural than a one-line fix.

## Open questions

- Should checkrun-only overrides (app-auth path, `pkg/plugins/override/override.go:576-597`) also get baseSHA-reuse treatment, or is item 2 scoped only to the status-based path? Not addressed by the issue text.
- Does anyone rely on Deck's job list showing override actions as a distinct job row (e.g. for auditing "who overrode what")? Worth a quick check before merging item 1, even though no code dependency was found.
