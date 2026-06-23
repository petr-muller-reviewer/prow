---
issue: 436
repo: kubernetes-sigs/prow
title: Prevent duplicate concurrent job reruns in Deck
verdict: LEGITIMATE
labels:
  - kind/feature
  - area/deck
  - area/prowcm
  - lifecycle/rotten
effort: "Level 2 — Moderate"
triaged_at: 2026-06-01T00:00:52Z
refresh_log:
  - at: 2026-05-31
    summary: Initial triage
  - at: 2026-06-01T00:00:52Z
    summary: "Labels updated: lifecycle/rotten and area/prowcm added; area/tide and help-wanted removed. Issue at auto-close risk."
---

# Triage: Issue #436

[kubernetes-sigs/prow#436](https://github.com/kubernetes-sigs/prow/issues/436) — **LEGITIMATE** | kind/feature | area/deck | area/prowcm | lifecycle/rotten | Level 2 — Moderate

## Summary

During the Kubernetes 1.33.0 release, multiple release managers independently triggered reruns of the same failed test (*Conformance-GCE-1.33-kubetest2*), resulting in 3 simultaneous runs wasting CI resources. The issue requests functionality to prevent duplicate concurrent reruns.

BenTheElder noted an existing queue feature (`job_queue_name` with capacity limits) that can enforce n=1 concurrency, but it's opt-in and not widely configured.

**Since previous triage (2026-05-31):**
- `lifecycle/rotten` label added — the issue has aged beyond stale and is at risk of automatic closure. Someone must comment or apply `lifecycle/frozen` to prevent it.
- `area/prowcm` label added — suggests a maintainer sees ProwController Manager as a relevant component alongside Deck.
- `area/tide` removed — consistent with the triage note that Tide handles merge automation and was not the right component label here.
- `help-wanted` not yet applied — the recommended label from the previous triage is still pending.

## Root Cause

**Race condition + lack of automatic deduplication**

1. **No deduplication at Deck level:** Each "Rerun" click creates a distinct ProwJob object (`cmd/deck/rerun.go:326`), each with a unique UID.
2. **Timing-dependent concurrency check:** Plank counts "pending or older triggered" jobs (`reconciler.go:1117-1139`). If two reruns arrive before the first transitions to PendingState, both pass the check.
3. **Queue feature not universally configured:** The existing queue mechanism is opt-in; the Conformance-GCE job had no queue set.

### Contributing Factors

- Multiple users with rerun permissions active simultaneously
- No UI feedback about pending reruns from other users
- Serialization locks only prevent same-thread races within the reconciler
- Brief TriggeredState → PendingState window allows duplicates through

## Architecture & Data Flow

```
User clicks Rerun → Deck validates auth → Deck creates ProwJob → K8s API (SchedulingState) → Plank reconciler → Concurrency check → Create Pod / leave scheduling
```

### Key Components

| Component | File | Role |
|-----------|------|------|
| Deck (Rerun UI) | `cmd/deck/rerun.go` | Handles rerun button, creates ProwJob |
| Plank (Scheduler) | `pkg/plank/reconciler.go` | Enforces concurrency limits, schedules jobs |
| Config | `pkg/config/config.go:722-728` | Defines `JobQueueCapacities` |
| ProwJob Spec | `pkg/apis/prowjobs/v1/types.go:228-235` | Defines `JobQueueName` field |

### Concurrency Control (existing)

- **Per-job:** `MaxConcurrency` field limits concurrent runs of a specific job
- **Per-queue:** `JobQueueCapacities` map limits concurrent jobs sharing a named queue
- Both are opt-in and require manual configuration

## Solution Approaches

### Recommended: Approach 2 — Rerun Deduplication in Deck

Add duplicate detection in `handleRerun()` before creating new ProwJob objects. Query for existing ProwJobs with the same name in SchedulingState, TriggeredState, or PendingState within a configurable time window (5–10 min). If found, return the existing job with UI feedback.

**Why this approach?**
- Prevents duplicates at the source (Deck), not downstream
- Immediate user feedback ("A rerun was already triggered 3 min ago by user X")
- Changes isolated to a single component
- High backwards compatibility, can be feature-flagged

**Implementation sketch:**
1. Add helper: `findRecentPendingRerun(ctx, jobName, within) (*ProwJob, error)`
2. Call it in `handleRerun()` before line 326 (job creation)
3. If duplicate found → return informative HTTP response
4. Optional: add configurable time window + per-job opt-out

### Alternatives Considered

**Approach 1: Automatic Queue Assignment with Smart Defaults**
Auto-assign all jobs to implicit queues by job name, default capacity 1.
- Pros: Zero config, prevents duplicates for all jobs, uses existing infrastructure.
- Cons: Changes default behavior, may delay legitimately parallel jobs, requires opt-out mechanism.
- Complexity: Medium.

**Approach 3: Global Default MaxConcurrency**
Add `default_max_concurrency_per_job_name: 1` config field applied to unconfigured jobs.
- Pros: Minimal code changes, simple config, uses existing checking.
- Cons: Doesn't prevent duplicate ProwJob creation, delayed user feedback, still creates jobs that sit in SchedulingState.
- Complexity: Low. Good as defense-in-depth alongside Approach 2.

**Approach 4: GUI Lock (not recommended)**
Disable rerun button temporarily after click. Does not solve the core issue — only prevents double-clicks from the same user, not multiple users triggering reruns independently.

## Effort Assessment

**Level 2 — Moderate** | help-wanted

| Factor | Rating | Notes |
|--------|--------|-------|
| Scope | 2–3 | 2–4 files, ~200–300 LOC |
| Complexity | 2 | Straightforward query + filter logic |
| Expertise | 2–3 | Go, client-go, ProwJob lifecycle |
| Clarity | 1–2 | Well-defined, minor config details to decide |
| Testing | 2–3 | Unit tests with mock K8s client |
| Backwards compat | 1–2 | Additive, feature-flaggable |
| Architecture fit | 2–3 | Natural extension of Deck rerun handler |

### Contributor Guidance

**Suitable for:** Contributors with solid Go experience and familiarity with Kubernetes client-go. Not a good-first-issue due to ProwJob lifecycle requirements.

**Preparation:**
1. Read `cmd/deck/rerun.go` — current rerun handler flow
2. Read `pkg/apis/prowjobs/v1/types.go` — ProwJob states
3. Read `cmd/deck/rerun_test.go` — existing test patterns
4. Reference `pkg/plank/reconciler.go:1117-1139` for state filtering patterns

**Open Questions for Maintainers:**
- Preferred time window default (5 min? 10 min?)
- Opt-in (feature flag) vs. opt-out initially?
- Add metrics for blocked duplicate reruns?
- Error message format for UI display?

## Labels & Title

| Action | Value | Rationale |
|--------|-------|-----------|
| Retitle | *Prevent duplicate concurrent job reruns in Deck* | Remove non-standard `[feature]` prefix; clarify component and behavior |
| `/area deck` | Already applied | — |
| `/help-wanted` | Add (pending) | Level 2 effort, suitable for skilled contributors |
| `kind/feature` | Already applied | — |
| `area/prowcm` | Already applied | Added by maintainer since initial triage |
| `area/tide` | Removed | Correctly removed since triage; Tide handles merge automation, not manual reruns |
| `lifecycle/rotten` | Currently applied — **urgent** | Issue at auto-close risk; apply `/lifecycle frozen` or post substantive comment to prevent closure |

## Draft GitHub Comment

```
/retitle Prevent duplicate concurrent job reruns in Deck

## Root Cause

This issue occurs due to a race condition combined with lack of automatic deduplication. When multiple users click "Rerun" within seconds of each other, Deck's rerun handler (`cmd/deck/rerun.go:245-347`) creates multiple distinct ProwJob objects, each with a unique ID. If these jobs reach Plank's scheduler before the first transitions from TriggeredState to PendingState, all pass the concurrency check and execute simultaneously. While BenTheElder mentioned the existing queue feature (`job_queue_name` with capacity limits), it's opt-in and not configured for most jobs, including the Conformance-GCE job from the incident.

## Recommended Solution

Add duplicate detection in Deck before creating new ProwJobs. When a rerun is requested, query for existing ProwJobs with the same name in SchedulingState, TriggeredState, or PendingState within a recent time window (5-10 minutes). If found, return the existing job instead of creating a duplicate, with UI feedback like "A rerun was already triggered 3 minutes ago by user X." This approach prevents duplicates at the source and provides immediate user feedback.

## Implementation Guidance

The fix should be implemented in `cmd/deck/rerun.go` by adding a helper function to query recent pending reruns before the job creation step at line 326. The implementation can reference `pkg/plank/reconciler.go:1117-1139` for ProwJob state filtering patterns. This is a moderate-complexity change affecting 2-4 files (~200-300 LOC) suitable for contributors familiar with Go and Kubernetes client-go. All triage research and detailed implementation guidance is available in the `issue-triage-436` branch.

/area deck
/help-wanted
```

## Key Files Reference

| File | Lines | What to look at |
|------|-------|----------------|
| `cmd/deck/rerun.go` | 245–347 | `handleRerun()` — where deduplication check goes |
| `cmd/deck/rerun.go` | 326 | ProwJob creation — insert check before this |
| `cmd/deck/rerun.go` | 151–162 | Existing ProwJob query by name pattern |
| `cmd/deck/rerun_test.go` | — | Test patterns to follow |
| `pkg/plank/reconciler.go` | 1117–1139 | ProwJob state filtering reference |
| `pkg/plank/reconciler.go` | 931–962 | `canExecuteConcurrentlyPerQueue()` |
| `pkg/plank/reconciler.go` | 909–929 | `canExecuteConcurrentlyPerJob()` |
| `pkg/config/config.go` | 722–728 | `JobQueueCapacities` definition |
| `pkg/apis/prowjobs/v1/types.go` | 228–235 | `JobQueueName` field |
| `pkg/apis/prowjobs/v1/types.go` | 174–179 | `MaxConcurrency` field |
