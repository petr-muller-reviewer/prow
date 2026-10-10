---
pr: kubernetes-sigs/prow#985
title: "horologium: reduce horologium's log noise"
head_sha: 443f3baf9390f3aad73a1f3b1b330e0e97f53dfa
base: main
reviewed_at: 2026-10-09T14:35:26Z
verdict: approve
refresh_log:
  - old_sha: 443f3baf9390f3aad73a1f3b1b330e0e97f53dfa
    new_sha: 443f3baf9390f3aad73a1f3b1b330e0e97f53dfa
    summary: "No code changes; recorded the October 7 /cc comment and review-request activity."
---
# Review

## Verdict

Approve with a non-blocking test suggestion. The three maintainer perspectives found no correctness or deployment blockers, and the maintenance burden is low. The scheduling and job creation paths remain unchanged while routine per-job log volume is reduced.

## What this PR does

- Returns per-sync counts from `sync` and attaches them to the existing summary log entry.
- Moves the per-periodic cron-skip and not-due messages to trace level.
- Removes a redundant condition and repeated fields from the cron-skip branch.
- Adds coverage for returned counter values and suppression of the per-job messages at debug level.

Since previous review:
- No code changes: the PR head remains `443f3baf9390f3aad73a1f3b1b330e0e97f53dfa`.
- `upodroid` commented `/cc @petr-muller` on October 7 at 15:35:55Z; GitHub recorded the mention and a review request to `petr-muller` shortly afterward. No inline comments, submitted reviews, or label changes were found.

## Findings

### [nit] Keep the trace logging path covered
- where: `cmd/horologium/main_test.go:675-680`
- concern: The test verifies that the messages are suppressed at debug level, but it would also pass if they were removed. Consider asserting that both messages are emitted when trace logging is enabled.
- excerpt: |
    // Per-job decisions must not be logged at debug or above: there is one
    // line per periodic per tick.
    for _, e := range hook.AllEntries() {
        if e.Message == "Skipping cron periodic" || e.Message == "Trigger time has not yet been reached." {
            t.Errorf("%q was logged at %s, expected trace", e.Message, e.Level)
        }
    }

## Checked

- **Code quality:** No critical issues or improvement requests; the scheduling and create decisions stay alongside their related counter updates.
- **Maintainability:** Low burden. The counter summary is cohesive; the trace-level assertion is a non-blocking suggestion.
- **Deployment risk:** Low. No config, API, RBAC, or scheduling changes; rollback requires reverting the binary. Operators will see aggregate counts instead of per-job skip details at info and debug levels.
- The two high-volume per-job messages use trace, and the once-per-tick summary includes the requested counters.
- `triggered` increments only after successful ProwJob creation; cron periodics that are not queued and evaluated periodics that are not created update their respective counters.
- Tests were not run as part of this review.

## Open questions

None.
