---
pr: kubernetes-sigs/prow#967
title: "override: allow overriding pending check runs, and cancel check run overrides"
head_sha: a8f462ce48984f843a7f4d4c5709fcae09ffb615
base: main
reviewed_at: 2026-10-01T13:59:46Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-01T14:00:17Z
  gated_head_sha: a8f462ce48984f843a7f4d4c5709fcae09ffb615
  reviewed_head_sha: a8f462ce48984f843a7f4d4c5709fcae09ffb615
refresh_log:
  - at: 2026-10-01T13:59:46Z
    old_sha: a8f462ce48984f843a7f4d4c5709fcae09ffb615
    new_sha: a8f462ce48984f843a7f4d4c5709fcae09ffb615
    summary: "No code changes; PR merged after an approval and a discussion of overriding the tide status."
---

# Review

## Gate

**Decision: merge.**

The reviewed head is the merged PR head, with no code changes after review. The only local risk item is the unvalidated live-GitHub behavior of updating a completed check run to `cancelled`; it is documented by the author, covered by the fake client, and failure is reported without silently claiming success. It is a rollout follow-up rather than a merge blocker. The only human feedback after review asked about overriding the `tide` status, and the author clarified that Tide does not use its own status when calculating mergeability; it raises no remaining code concern.

Gating list:

- `REVIEW.md` question at `pkg/plugins/override/override.go:843-851`: acceptable to merge; validate the completed-run cancellation transition against representative GitHub/GHES environments during rollout.

Independent merge risk:

- No API or configuration compatibility change was introduced. The intended behavioral change is limited to authorized `/override` users on App-authenticated installations: they can satisfy queued or in-progress check runs, and `/override-cancel` can restore a failed effective state. Deployments should validate the new GitHub Checks PATCH capability during rollout.

## Verdict

Approve with rollout suggestions.

The check-run override and cancellation paths are localized to the override plugin, verify Prow App ownership before mutating runs, and cover pending checks, identity matching, failures, ordering, and the complete lifecycle. The remaining live-GitHub validation gap and best-effort cancellation behavior are appropriate operational follow-ups, not merge blockers.

## What this PR does

- Makes queued and in-progress check runs eligible for `/override`, like pending commit statuses.
- Creates a successful same-name Prow check run when App authentication is available.
- Extends `/override-cancel` to find Prow-owned successful override check runs and update them to `cancelled`.
- Keeps status override cancellation and reports successful and failed individual cancellations together.
- Distinguishes already-passing contexts from unknown contexts in override replies.

Since previous review:

- No code changed; the reviewed head remains the merged PR head.
- A reviewer asked about overriding the `tide` context; the author clarified that Tide does not use its own status to calculate mergeability.
- Prucek approved the PR and it was merged.

## Findings

### [question] Validate cancellation against the live Checks API
- where: `pkg/plugins/override/override.go:843-851`
- excerpt: |
    cancelledCR := github.CheckRun{
        Status:     "completed",
        Conclusion: "cancelled",
        Output: github.CheckRunOutput{
            Title:   fmt.Sprintf("Prow override cancelled - %s", cr.Name),
            Summary: cancelDesc,
        },
    }
    if err := oc.UpdateCheckRun(org, repo, cr.ID, cancelledCR); err != nil {
- concern: The fake-client coverage is strong, but this is the first production use of this update path and the PR notes it has not been manually validated against GitHub. Please validate this completed-run transition on a representative GitHub or GHES staging installation before broad rollout.

## Checked

- Pending, queued, and in-progress check-run selection is deliberate; successful and neutral runs remain non-overridable.
- Check runs are listed before any cancellation write, so list failures leave statuses and check runs unchanged.
- Cancellation only mutates runs created by Prow's App, comparing immutable App ID first and using slug only when no ID is available.
- Tide maps completed `cancelled` check runs to failure, allowing deduplication to fall back to the original check result.
- No configuration schema, default, or API migration is introduced; non-App-auth installations retain status-only behavior.
- The working tree matches the reviewed PR head and `git diff --check` is clean.
- The PR merged at the reviewed head after approval.

## Open questions

- Has `PATCH /check-runs/{id}` with `conclusion: cancelled` been validated against the GitHub and GHES versions Prow supports? If so, please add the result to the PR before rollout.

## Followups

### Detect App-pinned required checks before overriding

```text
In kubernetes-sigs/prow, following merged PR #967 ("override: allow overriding pending check runs, and cancel check run overrides", merge commit 1bf709c1c93479bfb662fa070f5d967cfc13942b), investigate support for detecting App-pinned required checks before `/override` creates a Prow check-run override.

Inspect the GitHub and GHES APIs available to Prow for the App identity associated with each required check, and establish which supported server versions expose that data. If a compatible API is available across Prow's supported versions, extend the branch-protection/check-requirement model and the override plugin so `/override` detects a required check pinned to a different App and returns an explicit non-success explanation instead of reporting a successful override that cannot satisfy branch protection. Add focused unit tests for the model and plugin behavior, including a Prow-App match and a foreign-App mismatch.

Acceptance criteria:
- The investigation identifies the authoritative API and GHES compatibility constraints, with sources or tests in the change rationale.
- When Prow can identify an App-pinned required check belonging to a different App, `/override` does not claim that it has satisfied that requirement and tells the user why.
- Matching Prow-App requirements and existing unpinned requirements preserve current override behavior.
- Tests cover the new detection and compatibility behavior.

Out of scope: changing GitHub branch-protection configuration, adding a general ruleset-management feature, or attempting to bypass an App-pinned requirement.
```
