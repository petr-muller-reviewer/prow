---
issue: kubernetes-sigs/prow#436
title: "[feature] Rerun queue discard extra runs"
state: closed
labels: kind/feature, area/deck, area/prowcm, lifecycle/rotten
main_sha: dd066a6638b6304389e1ed251b91c07ca1b637ed
triaged_at: 2026-06-23T23:43:06Z
verdict: accepted
refresh_log:
  - at: 2026-05-31
    summary: Initial triage
  - at: 2026-06-01T00:00:52Z
    summary: "Labels updated: lifecycle/rotten and area/prowcm added; area/tide and help-wanted removed. Issue at auto-close risk."
  - at: 2026-06-23T23:43:06Z
    summary: "Issue auto-closed by k8s-triage-robot on 2026-06-23T08:30:11Z as not-planned due to inactivity. Full re-triage triggered; verdict remains accepted."
---

# Triage: kubernetes-sigs/prow#436

## Findings

### [reproducibility] Confirmed incident: 3 simultaneous reruns during k8s 1.33.0 release
- detail: Multiple release managers independently clicked "Rerun" on the same failed Conformance-GCE-1.33-kubetest2 job within seconds, producing 3 simultaneous runs and wasting CI resources.
- evidence: Issue body description of the 1.33.0 release cycle incident.

### [cause] No deduplication at Deck rerun handler
- detail: Each click of "Rerun" in Deck's UI calls `handleRerun()` which unconditionally creates a new ProwJob object with a unique UID. There is no prior-run check.
- evidence: `cmd/deck/rerun.go:245-347` — `handleRerun()` function; job creation at line 326.

### [cause] Timing-dependent concurrency check in Plank
- detail: Plank counts "pending or older triggered" jobs to enforce MaxConcurrency. If two reruns arrive before the first transitions from TriggeredState to PendingState, both pass the check and both execute.
- evidence: `pkg/plank/reconciler.go:1117-1139` — concurrency counting logic.

### [cause] Queue feature not universally configured
- detail: The existing `job_queue_name` + `JobQueueCapacities` mechanism can enforce n=1 concurrency per queue, but it is opt-in. The Conformance-GCE job had no queue set.
- evidence: `pkg/config/config.go:722-728` — `JobQueueCapacities` definition; `pkg/apis/prowjobs/v1/types.go:228-235` — `JobQueueName` field.

### [related-code] handleRerun — rerun entry point
- where: `cmd/deck/rerun.go:245-347`
- excerpt: |
    Job creation at line 326; no existing-run check before it. Query pattern for jobs by name exists at lines 151-162.

### [related-code] Plank concurrency checks
- where: `pkg/plank/reconciler.go:909-929` and `931-962`
- excerpt: |
    canExecuteConcurrentlyPerJob() (909-929) and canExecuteConcurrentlyPerQueue() (931-962) — reference these for state filtering patterns in any fix.

### [related-code] ProwJob types
- where: `pkg/apis/prowjobs/v1/types.go:174-179` and `228-235`
- excerpt: |
    MaxConcurrency field (174-179); JobQueueName field (228-235).

## Checked
- Deck rerun handler flow (`cmd/deck/rerun.go:245-347`)
- Plank concurrency enforcement logic (`pkg/plank/reconciler.go:1117-1139`)
- Existing queue/capacity mechanism (`pkg/config/config.go:722-728`)
- ProwJob state lifecycle and field types (`pkg/apis/prowjobs/v1/types.go`)
- Issue comments: BenTheElder noted `job_queue_name` as an opt-in workaround
- Issue auto-closure history: k8s-triage-robot closed as not-planned on 2026-06-23T08:30:11Z after lifecycle/rotten 30-day window expired
- No linked PRs or cross-references exist

## Next steps
- Reopen the issue: `/reopen` + `/remove-lifecycle rotten` to prevent immediate re-closure
- Post triage analysis comment (see draft below) with `/area deck`, `/help-wanted`
- Consider whether to apply `/lifecycle frozen` to prevent future bot interference while the issue is being actively worked
- Implement fix in `cmd/deck/rerun.go`: add `findRecentPendingRerun(ctx, jobName, within)` helper before the job creation at line 326; query for existing ProwJobs in SchedulingState, TriggeredState, or PendingState within a 5-10 minute window
- Decide open questions before implementation begins (see below)

## Open questions
- Preferred deduplication time window default: 5 min or 10 min?
- Opt-in (feature flag) vs. opt-out by default?
- Should duplicate-blocked reruns emit metrics?
- What HTTP response / UI message should the user see when a duplicate is blocked?
- Does `area/prowcm` (added by a maintainer) imply the fix should also touch the controller manager, or was it added as additional context?

## Draft comment for reopening

```
/reopen
/remove-lifecycle rotten

## Triage note

This is a valid feature request. The root cause is a race condition combined with absence of deduplication in Deck's rerun handler (`cmd/deck/rerun.go:245-347`): each "Rerun" click creates a new ProwJob unconditionally, and if two arrive before the first transitions from TriggeredState to PendingState, Plank's concurrency check passes both.

The existing `job_queue_name` mechanism (noted by BenTheElder) can enforce n=1 but is opt-in and not configured for most jobs.

**Recommended fix:** add a helper in `handleRerun()` that queries for existing ProwJobs with the same name in SchedulingState/TriggeredState/PendingState within a short window (5-10 min) before creating a new one. If found, return the existing job with informative UI feedback. Implementation touches 2-4 files (~200-300 LOC); suitable for contributors with Go and client-go experience.

/area deck
/help-wanted
```
