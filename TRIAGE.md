---
issue: kubernetes-sigs/prow#848
title: "status-reconciler can permanently miss config changes when delta delivery times out"
state: open
labels: (none)
main_sha: de7332b9aa43d1480db4c8099ca9ea7486b80743
triaged_at: 2026-08-17T22:57:43Z
verdict: accepted
advice:
  advised_at: 2026-08-21T16:00:57Z
  based_on_triaged_at: 2026-08-17T22:57:43Z
---

## Findings

### [reproducibility] Requires status-reconciler busy >1 minute during a config change
- detail: The drop only occurs if `status-reconciler` is still processing an earlier notification (e.g. listing PRs, GitHub API calls) when `config.Agent.Set()` tries to deliver a new `Delta` and the 1-minute per-subscriber timeout elapses. A second config change must land afterward for the loss to become permanent (its `Before` will reflect the dropped change).
- evidence: `pkg/config/agent.go:403-425`

### [cause] Agent.Set() mutates live config before attempting delivery, dropping timed-out deltas silently
- detail: `Set()` takes the lock, computes `oldConfig` from the current `ca.c`, builds `delta := Delta{oldConfig, *c}`, sets `ca.c = c` synchronously, then spawns a goroutine per subscriber that does `select { case sub <- delta: case <-end.C: }` with a 1-minute timer. The timeout branch drops `delta` with no logging. Because `ca.c` already advanced before delivery was attempted, any delta not delivered in time is unrecoverable.
- evidence: `pkg/config/agent.go:403-425`

### [cause] status-reconciler trusts delta.Before instead of tracking its own last-processed state
- detail: `Controller.Run()` calls `reconcile(change, log)` for each delta received on the subscription channel. `reconcile()` diffs `delta.Before.PresubmitsStatic` against `delta.After.PresubmitsStatic` (via `addedBlockingPresubmits`/`removedPresubmits`/`migratedBlockingPresubmits`) treating `Before` as authoritative. There is no independent "last processed" state on `Controller`, so a dropped delta's transition is never re-diffed by any later delta.
- evidence: `pkg/statusreconciler/controller.go:161-209`

### [related-code] SetWithoutBroadcast — existing pattern for consuming without re-triggering
- where: `pkg/config/agent.go:433-436`
- excerpt: |
    func (ca *Agent) SetWithoutBroadcast(c *Config) {
    	ca.mut.Lock()
    	defer ca.mut.Unlock()
    	ca.c = c
    }
- relevance: Not currently used by status-reconciler, but already used by `pkg/moonraker/moonraker.go:134` as the mechanism a subscriber uses to update its own agent without re-broadcasting. Relevant prior art for the fix pattern.

### [related-code] Other DeltaChan subscribers use overwrite semantics, not vulnerable to this bug
- where: `pkg/moonraker/moonraker.go:107-134`, `pkg/pubsub/subscriber/server.go:158-159`
- excerpt: |
    (both subscribers apply event.After wholesale as their new live config,
    rather than diffing event.Before against independently tracked state)
- relevance: Confirms status-reconciler is the only subscriber exposed to the "lost intermediate diff" failure mode; blast radius is contained to status-reconciler.

### [related-code] No unit tests exist for Agent.Set concurrency/timeout/drop behavior
- where: `pkg/config/agent.go` (no corresponding `agent_test.go`)
- excerpt: |
    (file does not exist)
- relevance: The issue's acceptance criteria require a test demonstrating timeout-then-recover behavior; no scaffolding currently exists for it.

### [related-code] Existing controller tests never exercise Run() or the real channel
- where: `pkg/statusreconciler/controller_test.go:762-1309`
- excerpt: |
    func TestControllerReconcile(t *testing.T) {
    	// constructs config.Delta by hand and calls c.reconcile(delta, ...) directly
    }
- relevance: `TestControllerReconcile` bypasses `Run()` and the subscription channel entirely, so dropped/timeout scenarios are untested.

### [related-issue] kubernetes-sigs/prow#841
- ref: kubernetes-sigs/prow#841
- relevance: Related but distinct — covers orphaned required-contexts (`migrator.RetireMode`/`stateAny`, `migrator.go:171-174`), a Tide `requirementDiff` description-shadowing bug (`pkg/tide/status.go:265-267`), and GitHub-ruleset context retirement. Does not describe the dropped-delta/timeout mechanism; #848 correctly isolates itself from #841's ruleset/Tide items, which remain out of scope here.

### [related-pr] kubernetes-sigs/prow#849
- ref: kubernetes-sigs/prow#849
- relevance: Open PR ("fix: status-reconciler track own config state for dropped deltas") already implementing the issue's proposed fix — `lastProcessed` config tracked by `Controller`, diffed against live `Config()` instead of `delta.Before`, plus a `logrus.Warn` in `Agent.Set()`'s timeout branch. State: mergeable, CI passing, not yet `lgtm`/`approved`. Gap: `lastProcessed` advances after `reconcile()` regardless of its returned error, which could silently skip retrying a genuinely failed reconciliation — flagged as a review comment, not separate issue scope.

## Checked

- Confirmed `Agent.Set()` and `Controller.reconcile()` mechanics against current `main` (`de7332b9a`) match the issue's description exactly.
- Checked all `DeltaChan` subscribers in the repo (`moonraker`, `pubsub/subscriber`) for the same vulnerability class; only `status-reconciler` is exposed.
- Checked git history on `agent.go`'s `Set` and `controller.go`'s `reconcile` — no prior related fix attempts.
- Checked for existing tests covering dropped/timeout delta scenarios — none exist (`pkg/config/agent_test.go` doesn't exist; `controller_test.go` never exercises `Run()`).
- Confirmed PR #849 is open, mergeable, and CI-passing as of this triage.
- Compared #848 against #841 to confirm correct issue scoping/no duplication.

## Next steps

- Review PR #849 (https://github.com/kubernetes-sigs/prow/pull/849), specifically flagging that `lastProcessed` advances unconditionally after `reconcile()` even when it returns an error.
- Confirm or request tests covering the issue's four acceptance criteria: (1) a dropped/timed-out delta followed by a delta that still reconciles the skipped transition, (2) a newly added blocking presubmit still eventually triggers on open PRs after a dropped notification, (3) the new warning log carries enough context to diagnose which delta was dropped, (4) normal processing doesn't reconcile the same transition twice.
- Cross-link #848 and #849 if not already referenced from one another.
- Apply labels manually: `kind/bug`, `area/status-reconciler`, `priority/important-soon`.

## Advice

- action: Apply labels to the issue — none of the recommended labels are currently applied (`gh issue view 848` shows `labels: []`).
  rationale: Triage recommended `kind/bug`, `area/status-reconciler`, `priority/important-soon`; all three exist in the repo's label set and none are set yet.
  command: |
    gh issue edit 848 --repo kubernetes-sigs/prow --add-label "kind/bug,area/status-reconciler,priority/important-soon"

- action: Comment on the issue linking PR #849, since a fix is already in flight and its body already contains "Fixes #848".
  rationale: Prevents duplicate contributor effort; PR #849 (Prucek) implements the proposed approach and will auto-close this issue on merge.
  command: |
    gh issue comment 848 --repo kubernetes-sigs/prow --body "Fix in flight: #849 implements the lastProcessed/live-Config() diff approach and the Agent.Set() drop-warning described above. Routing review effort there rather than duplicating it here."

- action: Leave a PR review comment on #849 flagging that \`lastProcessed\` advances even when \`reconcile()\` returns an error.
  rationale: Confirmed by code research (see Findings: related-pr kubernetes-sigs/prow#849) — this is a correctness gap worth resolving before merge, not new issue scope.
  command: |
    gh pr comment 849 --repo kubernetes-sigs/prow --body "lastProcessed appears to advance unconditionally after reconcile() is called, even when it returns an error — this would silently skip retrying a transition whose reconciliation genuinely failed. Should this only advance on a nil error?"

- action: No action needed on PR approval routing — already tracked by the bot.
  rationale: PR #849 is approved by kaovilai (issue author) and self-approved by Prucek (PR author) but not yet \`lgtm\`'d; the automated OWNERS bot comment already identifies cblecker as the required approver for \`pkg/OWNERS\`, so no manual assignment command is needed.

## Open questions

- Should `lastProcessed` only advance after `reconcile()` returns `nil`, or does the codebase have a different intended retry story for partial/failed reconciliation? Worth resolving on PR #849 before merge rather than as new issue scope.
