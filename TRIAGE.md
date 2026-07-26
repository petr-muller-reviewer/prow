---
issue: kubernetes-sigs/prow#768
title: "Support sticky overrides via /override-sticky command"
state: open
labels: ""
main_sha: 1e6ed72765e2d88f3bb1e06af85720e88b5db3af
triaged_at: 2026-06-24T01:02:30Z
verdict: accepted
---

## Findings

### [cause] Override status description lacks baseSHA — Tide cannot validate across base advances
- detail: `/override` sets status description to `"Overridden by <user>"`. `prowJobsFromContexts()` only promotes statuses to synthetic ProwJobs when `BaseSHAFromContextDescription(description)` returns a non-empty SHA matching the current subpool's `sp.sha`. The plain override description returns `""`, so the override result is never promoted.
- evidence: `pkg/plugins/override/override.go:290-292` (description func), `pkg/tide/tide.go:1050-1055` (baseSHA check in prowJobsFromContexts)

### [cause] Override ProwJob carries old baseSHA — excluded when base branch advances
- detail: The override plugin creates a ProwJob with `SuccessState` using the baseSHA at override time. When another PR merges and `sp.sha` changes, `accumulate()` filters ProwJobs by `pj.Spec.Refs.Pulls[0].SHA == pr.HeadRefOID` and implicitly by the subpool's baseSHA scope. The old-baseSHA override ProwJob is no longer considered, so the context becomes "missing" and Tide triggers a new run.
- evidence: `pkg/plugins/override/override.go:494-517` (ProwJob creation), `pkg/tide/tide.go:1089-1116` (accumulate missing logic)

### [cause] No persistent "sticky" intent mechanism exists
- detail: Override is implemented purely as a GitHub status + ProwJob. There is no label, annotation, or other durable record of intent that survives Tide's re-evaluation cycle.
- evidence: `pkg/plugins/override/override.go:324-556` (entire handle function)

### [related-code] ContextDescriptionWithBaseSha / BaseSHAFromContextDescription
- where: `pkg/config/config.go:3416-3458`
- excerpt: |
    func ContextDescriptionWithBaseSha(humanReadable, baseSHA string) string {
        // embeds " BaseSHA:<40-char-sha>" at end of description
    }
    func BaseSHAFromContextDescription(description string) string {
        split := strings.Split(description, contextDescriptionBaseSHADelimiter)
        if len(split) != 2 || len(split[1]) != 40 { return "" }
        return split[1]
    }

### [related-code] prowJobsFromContexts — synthetic ProwJob promotion
- where: `pkg/tide/tide.go:1043-1073`
- excerpt: |
    if baseSHAForContext := config.BaseSHAFromContextDescription(string(headContext.Description));
        baseSHAForContext != "" && baseSHAForContext == baseSHA {
        passingCurrentContexts = append(...)
    }

### [related-code] accumulate — determines missing/pending/success per PR
- where: `pkg/tide/tide.go:1077-1135`
- excerpt: |
    if s, ok := psStates[ps.Context]; !ok {
        missingTests[pr.Number] = append(missingTests[pr.Number], ps)
    } else if s == failureState {
        missingTests[pr.Number] = append(missingTests[pr.Number], ps)
    }

### [related-issue] Prior discussion of Tide re-running overridden jobs
- ref: kubernetes-sigs/prow#238
- relevance: Original 2024 report of the same behavior; closed by reporter in April 2025 as "no longer affecting us." My 2024 comment there identified the two override motivations (broken job vs. reviewed-and-accepted) and the tension between them.

## Checked

- Override plugin creates both a GitHub status AND a ProwJob; neither survives a baseSHA change
- `description()` in override plugin returns `"Overridden by <user>"` with no baseSHA encoding — confirmed no `ContextDescriptionWithBaseSha` call
- Checkrun-based overrides (GitHub Apps auth path) follow the same pattern — no baseSHA embedded in the override checkrun output (`pkg/plugins/override/override.go:530-554`)
- No existing sticky-override or label-based override persistence anywhere in the codebase
- #238 was closed by the original reporter, not resolved with a code fix

## Next steps

- Apply labels: `kind/feature`, `area/plugins`, `area/tide`, `help-wanted`
- Post comment linking to #238 for historical context
- If a contributor engages: require design sketch covering (a) label naming scheme for encoding context names with special chars, (b) whether sticky override should cancel on new push to PR branch, (c) race condition handling between Tide trigger and re-override firing — before any code is written
- Evaluate whether Approach 2 (Tide modification to skip triggering sticky-overridden contexts) is worth a separate design discussion with maintainers; it's more efficient but touches Tide's core safety logic

## Open questions

- Should sticky overrides auto-cancel when the PR head SHA changes (new push to PR branch), or persist? Issue doesn't address this; canceling on head-change seems safer.
- What happens to sticky labels if the overridden job is later removed from the required list? Need a cleanup mechanism or orphan handling.
- Is GitHub PR label storage sufficient for encoding arbitrary context names (spaces, slashes), or does this push toward a different state mechanism (e.g., structured bot comment)?
- Is Approach 2 (Tide modification) acceptable given the override↔Tide coupling it introduces, given the efficiency benefit of not triggering the job at all?
