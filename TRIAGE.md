---
issue: kubernetes-sigs/prow#848
title: "status-reconciler can permanently miss config changes when delta delivery times out"
state: open
labels: kind/bug, area/status-reconciler
main_sha: da76fd4bf78649f471b25f7cd438d554c7626e7e
triaged_at: 2026-08-27T14:21:29Z
verdict: accepted
---

## Findings

### [cause] Agent.Set() mutates live config before attempting delivery, dropping timed-out deltas silently
- detail: `Set()` sets `ca.c = c` synchronously, then spawns a goroutine per subscriber doing `select { case sub <- delta: case <-end.C: }` with a 1-minute timer. Timeout drops `delta` silently. Because `ca.c` already advanced, a dropped delta's transition is unrecoverable through the agent alone.
- evidence: `pkg/config/agent.go:403-425`

### [cause] status-reconciler trusts delta.Before instead of tracking its own last-processed state
- detail: `reconcile()` diffs `delta.Before.PresubmitsStatic` against `delta.After.PresubmitsStatic`, treating `Before` as authoritative, with no independent "last processed" state on `Controller`. A dropped delta's transition is never re-diffed by any later delta.
- evidence: `pkg/statusreconciler/controller.go:161-209`

### [related-pr] kubernetes-sigs/prow#849 — original fix, held in review
- ref: kubernetes-sigs/prow#849
- relevance: Consumer tracks `lastProcessed`, diffs against live `Config()` instead of `delta.Before`. 66-line diff, no tests. Held by maintainer review (petr-muller, https://github.com/kubernetes-sigs/prow/pull/849#pullrequestreview-4996069341): "degrading the delta channel to be just a notification mechanism for the destination to do its own config fetch feels like it breaks a fairly nice abstraction and separation of concern." Effectively superseded by #870/#871, which were written specifically because this approach was found lacking. State: `do-not-merge/hold`, no lgtm.

### [related-pr] kubernetes-sigs/prow#870 — fix at the source, breaking change
- ref: kubernetes-sigs/prow#870
- relevance: Makes `config.Agent.Subscribe()`/`Set()` lossless via single-slot coalescing at the source — `Subscribe()` changes from `Subscribe(subscription DeltaChan)` to agent-owned `Subscribe() DeltaChan` (receive-only, coalescing); removes the goroutine/timeout mechanism entirely; fixes the bug for every subscriber, not just status-reconciler. 236-line diff, 2 new tests in `pkg/config/agent_test.go` covering the coalescing mechanism directly. Breaking API change for external vendorers of `pkg/config.Subscribe` — Prucek (comment on PR): "this is a breaking API change for anyone vendoring pkg/config and calling Subscribe directly... I checked ci-tools and it doesn't call Subscribe... but I haven't checked other consumers... /hold" (gave `lgtm`+`hold` together). Currently failing `pull-prow-verify-lint` and `pull-prow-unit-test-race-detector-nonblocking`; also flagged `do-not-merge/invalid-commit-message` (commit message used unsupported "(fix #848)" format).

### [related-pr] kubernetes-sigs/prow#871 — decouple + retry, non-breaking
- ref: kubernetes-sigs/prow#871
- relevance: Dedicated goroutine drains the delta channel immediately on the consumer side (so `Agent.Set()`'s sender never blocks/drops), tracking latest `After`; `lastReconciled` advances only after a successful `reconcile()`. Adds bounded retry (`defaultMaxReconcileAttempts = 5`, `defaultReconcileRetryDelay = 30s`, not CLI-configurable) so a "poison" config transition can't wedge the controller forever. 394-line diff, includes a realistic drop-simulation test (`TestControllerRunRecoversDroppedDelta`: sends v1→v2 then jumps straight to v3→v4, asserts presubmit still triggers) and a poison-config no-wedge test. Non-breaking. Bundles an unrelated drive-by fix to `continueOnError` handling in `pjutil.FilterPresubmits` inside `triggerNewPresubmits` — scope creep unrelated to #848. Currently failing the same two CI checks as #870; not yet `lgtm`'d.

### [related-issue] kubernetes-sigs/prow#841
- ref: kubernetes-sigs/prow#841
- relevance: Related but distinct — covers orphaned required-contexts, a Tide `requirementDiff` description-shadowing bug, and GitHub-ruleset context retirement. Does not describe the dropped-delta/timeout mechanism.

## Checked

- Re-confirmed root cause against `main_sha` `da76fd4bf7` — unchanged from first triage pass.
- Read full diffs of #849, #870, #871 (`gh pr diff`) — each PR description accurately reflects its actual diff.
- Checked all in-repo callers of `config.Agent.Subscribe(` (moonraker, pubsub/subscriber, statusreconciler) — all updated correctly in #870's diff; no other in-repo direct callers exist.
- Checked #870's flagged breaking change against Prucek's PR comment text directly.
- Checked #871's retry defaults and confirmed they are not exposed via CLI flags.
- Checked #871's `continueOnError` fix against `pjutil.FilterPresubmits` call site — confirmed unrelated to #848's dropped-delta mechanism.
- Checked current CI/review/label state on all three PRs.
- Confirmed issue #848 has `kind/bug`, `area/status-reconciler` applied, but not `priority/important-soon`.

## Next steps

- Decide between #870 (source fix, breaking) and #871 (consumer fix, non-breaking, better tested) — maintainer judgment call, not further research.
- Investigate why both #870 and #871 fail the same two CI checks (`pull-prow-verify-lint`, `pull-prow-unit-test-race-detector-nonblocking`) before treating it as a defect in either.
- If leaning #870: resolve `do-not-merge/invalid-commit-message`, seek broader confirmation no external vendorer depends on the current `Subscribe(chan<- Delta)` signature.
- If leaning #871: ask author to split the `continueOnError` fix into its own PR; consider whether retry parameters should be configurable.
- Close or mark #849 as superseded once a direction is chosen.
- Apply `priority/important-soon` label to #848 (still missing).

## Open questions

- Fix it once at the source for every subscriber and accept a breaking change (#870), or fix it for this one consumer without breaking anything (#871) — which is the preferred general design stance?
- Is there an actual external consumer of `config.Agent.Subscribe` outside this repo that would be broken by #870?
- Should #871's retry count/delay (5 attempts, 30s) be configurable, or are the hardcoded defaults acceptable long-term?
