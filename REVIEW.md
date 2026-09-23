---
pr: kubernetes-sigs/prow#967
title: "override: allow overriding pending check runs, and cancel check run overrides"
head_sha: a8f462ce48984f843a7f4d4c5709fcae09ffb615
base: main
reviewed_at: 2026-09-23T21:23:09Z
verdict: approve
---

# Review

## Verdict

Approve with rollout suggestions.

The check-run override and cancellation paths are localized to the override plugin, verify Prow App ownership before mutating runs, and cover pending checks, identity matching, failures, ordering, and the complete lifecycle. The remaining live-GitHub validation gap and best-effort cancellation behavior are appropriate operational follow-ups, not merge blockers.

## What this PR does

- Makes queued and in-progress check runs eligible for `/override`, like pending commit statuses.
- Creates a successful same-name Prow check run when App authentication is available.
- Extends `/override-cancel` to find Prow-owned successful override check runs and update them to `cancelled`.
- Keeps status override cancellation and reports successful and failed individual cancellations together.
- Distinguishes already-passing contexts from unknown contexts in override replies.

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

## Open questions

- Has `PATCH /check-runs/{id}` with `conclusion: cancelled` been validated against the GitHub and GHES versions Prow supports? If so, please add the result to the PR before rollout.
