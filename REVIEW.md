---
pr: kubernetes-sigs/prow#985
title: "horologium: reduce horologium's log noise"
head_sha: 443f3baf9390f3aad73a1f3b1b330e0e97f53dfa
base: main
reviewed_at: 2026-10-15T23:14:37Z
verdict: approve
refresh_log:
  - old_sha: 443f3baf9390f3aad73a1f3b1b330e0e97f53dfa
    new_sha: 443f3baf9390f3aad73a1f3b1b330e0e97f53dfa
    summary: "No code changes; recorded the October 7 /cc comment and review-request activity."
  - old_sha: 443f3baf9390f3aad73a1f3b1b330e0e97f53dfa
    new_sha: 443f3baf9390f3aad73a1f3b1b330e0e97f53dfa
    summary: "No code changes; recorded @petr-muller's approval and the PR merge."
gate:
  decision: merge
  gated_at: 2026-10-10T15:39:44Z
  gated_head_sha: 443f3baf9390f3aad73a1f3b1b330e0e97f53dfa
  reviewed_head_sha: 443f3baf9390f3aad73a1f3b1b330e0e97f53dfa
---
# Review

## Gate

**Decision: MERGE**

The PR head is unchanged from the reviewed SHA. The only local review finding is a non-blocking test suggestion, and no substantive external feedback or hold was present at gate time. The diff changes logging behavior as intended but does not change Prow's APIs, configuration, or scheduling behavior.

### Findings disposition

- **REVIEW.md — Keep the trace logging path covered** (`cmd/horologium/main_test.go:675-680`): not addressed, non-gating. The test only checks suppression at debug, while the implementation logs both messages at trace in `cmd/horologium/main.go:221,225`. Consider adding a trace-level assertion; this coverage improvement does not need to block merge.
- No substantive inline review comments or submitted reviews were present at gate time. The `/cc @petr-muller` issue comment is not actionable feedback.

### Gating list

None.

### Independent merge risk

No notable merge risk. The only operator-visible behavior change is the intended reduction of per-job log detail at info/debug levels; the existing sync summary now carries aggregate counts, and trace logging retains per-job messages. `sync` is unexported within `cmd/horologium`; no external API or configuration schema changes are present.

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

Since previous review (2026-10-09):
- No code changes: the PR head remains `443f3baf9390f3aad73a1f3b1b330e0e97f53dfa`.
- `petr-muller` submitted an APPROVED review on October 10 at 15:46:14Z; `kubernetes-prow[bot]` posted the approval notification eight seconds later.
- `app/kubernetes-prow` merged the PR on October 10 at 16:07:17Z. No inline review comments were added.

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

## Followups

### Probe: Verify trace-level diagnostics remain available

- **Category:** tests
- **Mode:** probe
- **Where:** `cmd/horologium/main.go:221,225`; `cmd/horologium/main_test.go:675-680`
- **Necessity:** could — protects the trace diagnostic path promised by PR #985.
- **Why followup:** The merged code calls `Trace`, but the test only checks debug suppression. The probe was deferred from the merge because no runtime defect was identified.
- **Handoff prompt:**

```text
In kubernetes-sigs/prow, following PR #985, “horologium: reduce horologium's log noise” (merge commit d3dc1c7b61b3f489e1f2d0f65b62b18dc6d8d079), probe the trace logging behavior in cmd/horologium/main.go and cmd/horologium/main_test.go, especially TestSyncStatsAndPerJobLogLevel.

Task: Add temporary tests that exercise both per-job messages at Debug and Trace levels. Confirm both messages are suppressed at Debug and emitted at Trace. Run `go test ./cmd/horologium` and classify each failure as either a real implementation bug or a test/setup mistake.

Acceptance criteria: Both levels are exercised and every failure is classified with evidence. If a real implementation bug is found, apply only the minimal fix and report the failing case. If no bug is found, leave no files changed.

Scope guard: Delete all probe tests before finishing. Do not change scheduling, counters, or unrelated code. The tests are a means to find a bug, not a deliverable.
```

### Probe: Verify sync statistics after partial create failures

- **Category:** tests
- **Mode:** probe
- **Where:** `cmd/horologium/main.go:244-255`; `cmd/horologium/main_test.go:621-681`
- **Necessity:** could — checks that the new operational count stays accurate during partial API failures.
- **Why followup:** The counter is documented as successful ProwJob creations, but the added test covers only successful creates. The merged implementation increments after successful `Create` and reports errors separately, so this was not a merge blocker.
- **Handoff prompt:**

```text
In kubernetes-sigs/prow, following PR #985, “horologium: reduce horologium's log noise” (merge commit d3dc1c7b61b3f489e1f2d0f65b62b18dc6d8d079), probe syncStats error-path behavior in cmd/horologium/main.go and cmd/horologium/main_test.go.

Task: Create a temporary client wrapper that fails Create for one periodic ProwJob and delegates Create for another. Exercise sync and check that it returns an error, continues processing subsequent periodics, and reports `stats.triggered` as only the number of successful creates. Run `go test ./cmd/horologium` and classify failures as real implementation bugs or test/setup mistakes.

Acceptance criteria: The partial-failure path is exercised and each failure is classified with evidence. If a real bug is found, apply only the minimal fix and report the failing case. If no bug is found, leave no files changed.

Scope guard: Delete all probe tests before finishing. Do not change normal scheduling behavior or unrelated code. The tests are a means to find a bug, not a deliverable.
```
