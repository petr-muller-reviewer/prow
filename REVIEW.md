---
pr: kubernetes-sigs/prow#951
title: "Map scheduling ProwJob state to GitHub pending status"
head_sha: 3e50f77192258c8513d2664cbd103051b6b8d272
base: main
reviewed_at: 2026-09-21T23:45:23Z
verdict: approve
---

## Verdict

Approve. This small, localized fix maps the previously unhandled scheduling ProwJob state to the existing GitHub `pending` status and covers the externally visible status-reporting behavior. It makes no configuration, API, dependency, or permission changes.

## What this PR does

- Treats `prowapi.SchedulingState` as GitHub `pending` in the GitHub reporter.
- Keeps scheduled-but-not-yet-running jobs in the same non-terminal status category as triggered and pending jobs.
- Adds a regression case that asserts the pending status emitted for a reporting presubmit job in scheduling state.

## Findings

None.

## Checked

- `pkg/github/report/report.go:56-73`: the added lifecycle-state mapping uses the existing `github.StatusPending` constant and preserves the established translation boundary.
- `pkg/github/report/report_test.go:323-330`: table-driven regression coverage asserts the externally visible GitHub status for `SchedulingState`.
- Configuration, CRD/API, authorization, and tenant-routing compatibility: unchanged.
- Operational impact: status creation remains gated by normal reporting conditions; the additional update is bounded and deduplicated per state transition.

## Open questions

None for this PR.

## Followups

### [could] Include SchedulingState in GetAllProwJobStates
- where: `pkg/apis/prowjobs/v1/types.go`, Gerrit reporter callers
- rationale: The helper's stated all-states contract may omit `SchedulingState`, allowing similar state-list drift outside the GitHub reporter.
- handoff: |
    In kubernetes-sigs/prow, inspect `GetAllProwJobStates()` in `pkg/apis/prowjobs/v1/types.go` and every caller that depends on its returned lifecycle-state list, especially Gerrit reporting. Add `SchedulingState` only where the helper's all-states contract requires it, and add or update focused tests proving the caller accepts the scheduling state. Keep this separate from GitHub reporter behavior and avoid changing status mappings beyond the required state-list correction.
