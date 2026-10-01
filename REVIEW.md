---
pr: kubernetes-sigs/prow#751
title: "Add retry logic to handle successive failure attempts"
head_sha: 6d527d4bcd6f273d4f3f0a563702a1502238dfb2
base: main
reviewed_at: 2026-06-15T10:41:31Z
verdict: approve
---

## Findings

### [should-fix] No retry logging
- where: `pkg/jira/jira.go:533-549`
- concern: The retry loop silently removes fields and retries up to 5 times. No log trail shows which fields were removed or how many retries occurred. Flagged independently by all three review perspectives (code quality, maintainability, deployment risk). A `logrus` line per iteration with attempt number and removed field names would significantly improve production debuggability.
- excerpt: |
    for range 5 {
        createdIssue, err = jc.CreateIssue(childIssue)
        if err == nil {
            break
        }
        ...
        childIssue, newErr = unsetProblematicFields(childIssue, JiraErrorBody(err))

### [should-fix] No test coverage for CloneIssue
- where: `pkg/jira/jira.go:513-574`
- concern: Zero tests for `CloneIssue` exist in `jira_test.go`. Pre-existing debt, but the new retry loop introduces a non-trivial state machine (success, non-400 error, fixable 400, unfixable 400, retry exhaustion) that warrants at least one table-driven test with a fake `Client` returning different 400 errors on successive calls. Flagged by code quality and maintainability reviewers.

### [nit] Magic retry cap
- where: `pkg/jira/jira.go:533`
- concern: The retry limit `5` is a bare literal. A named constant (`maxCloneRetries`) would make intent self-documenting and the value easier to find/change. Flagged by code quality and maintainability reviewers.
- excerpt: |
    for range 5 {

### [nit] Wasted work on final iteration
- where: `pkg/jira/jira.go:533-549`
- concern: On the 5th (final) iteration, if `CreateIssue` returns a 400 and `unsetProblematicFields` succeeds, the corrected issue is never retried — the loop ends and the error is returned. The last `unsetProblematicFields` call does JSON marshal/unmarshal work that is thrown away. Not a correctness bug (the function correctly returns the error), but wasteful. Could guard the unset call or use `range 6`.
- excerpt: |
    for range 5 {
        createdIssue, err = jc.CreateIssue(childIssue)
        if err == nil {
            break
        }
        // on iteration 5: unsetProblematicFields runs but loop ends without retry

### [question] No-progress retry scenario
- where: `pkg/jira/jira.go:533-549`
- concern: If Jira returns a 400 with an empty `Errors` map or fields not present in the issue, `unsetProblematicFields` succeeds but changes nothing. The loop resubmits the same payload up to 5 times. Pre-existing issue amplified from 2 to 5 wasted calls. A check that `processedResponse.Errors` is non-empty before retrying would guard against this.

## Checked
- `for range 5` is idiomatic Go 1.22+ — Prow targets recent Go
- `err` variable scoping: declared on line 516 via `data, err := ...`, reused via assignment (not `:=`) in the loop — correct
- Early-return paths preserved inside the loop: non-400 errors and `unsetProblematicFields` failures return immediately
- Post-loop `if err != nil` correctly catches retry exhaustion
- `unsetProblematicFields` correctly deletes reported fields from the issue map and strips null customfields
- No configuration, API, or interface changes — purely internal implementation
- No callers of `CloneIssue` outside the jira package — minimal blast radius
- Deployment risk is LOW: no config migration, no ordering dependencies, safe to roll back

## Open questions
- Should a log line be added when retrying, including the attempt number and which fields were unset?
- Is there a known upper bound on how many distinct problematic fields Jira reports across successive attempts? Is 5 retries based on observed behavior?
- Should the loop guard against the no-progress case where `unsetProblematicFields` removes nothing?
