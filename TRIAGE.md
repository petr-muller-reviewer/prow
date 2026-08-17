---
issue: kubernetes-sigs/prow#841
title: "`status-reconciler`: a required context that never once reported can never be retired, permanently deadlocking Tide under `github_merge_blocks_policy: block`"
state: closed
labels:
triaged_at: 2026-08-17T22:18:10Z
main_sha: de7332b9aa43d1480db4c8099ca9ea7486b80743
verdict: duplicate
---

## Findings

### [reproducibility] Original scenario is a GitHub ruleset, not Prow-managed protection
- detail: The reporter's real-world trigger (`openshift/oadp-operator#2346`) is blocked by a GitHub ruleset-pinned required context, not a branch-protection-managed one. branchprotector only speaks the legacy Branch Protection API and never touches rulesets, so this class of blockage is out of scope for any Prow component, not a `status-reconciler` bug.
- evidence: petr-muller comment, https://github.com/kubernetes-sigs/prow/issues/841#issuecomment-5259623345

### [cause] `stateAny` retire condition requires a pre-existing status entry
- detail: `Mode.processStatuses` requires the context to already be present in the PR's combined status for a `stateAny` condition to match. A context that never once posted has no entry, so `retireAction` never fires — confirmed accurate, but not the operative cause of the reporter's cited incident (see reproducibility finding above).
- evidence: `pkg/statusreconciler/migrator/migrator.go:171-174`

### [cause] `Agent.Set` silently drops a config delta on subscriber timeout — the real bug
- detail: Each subscription's delta is sent over a channel guarded by a 1-minute timer; if the send doesn't complete in time, the event is dropped (`case <-end.C:`) while `ca.c` has already been updated synchronously. `status-reconciler` trusts `delta.Before` as its diff baseline instead of tracking its own last-processed config, so a dropped delta's changes (e.g. a newly added blocking presubmit) are never diffed against and `triggerNewPresubmits` never runs for them. Identified by Prucek (MEMBER), verified by direct code read.
- evidence: `pkg/config/agent.go:403-421`; https://github.com/kubernetes-sigs/prow/issues/841#issuecomment-5281038208

### [related-code] Tide's block-policy status message is gated behind other reasons — working as intended
- detail: `requirementDiff` only reports the `github_merge_blocks_policy: block` description if `desc == ""`, and it's the last of ~7 gates checked. Confirmed accurate; judged intentional given status length limits and that earlier reasons (missing label, failing test) are usually more actionable for the author than a GitHub-side block they can't self-resolve.
- where: `pkg/tide/status.go:265-267`

### [related-issue] Real bug split out to a dedicated issue
- ref: kubernetes-sigs/prow#848
- relevance: "status-reconciler can permanently miss config changes when delta delivery times out" — open, correctly scoped to the `Agent.Set`/delta-drop bug found in this thread. This is where the actionable fix belongs.

### [related-pr] Real-world trigger PR cited by reporter
- ref: openshift/oadp-operator#2346
- relevance: PR whose `mergeStateStatus` stayed `BLOCKED` due to two never-triggered required contexts; motivated this issue. Root blockage was a GitHub ruleset (openshift/release#82902 set `github_merge_blocks_policy: block`), not a Prow bug.

## Checked

- `pkg/statusreconciler/migrator/migrator.go:171-174` (`processStatuses`, `stateAny` case) matches the issue's description of the retire-blind-spot.
- `pkg/tide/status.go:265-267` (`requirementDiff` gate ordering) matches the issue's description; behavior judged intentional.
- `pkg/config/agent.go:403-421` (`Agent.Set`) matches Prucek's delta-drop description exactly.
- Issue state: closed 2026-08-13T16:40:07Z, `stateReason: DUPLICATE`, cross-referenced by #848 (open, correctly scoped to the real bug).

## Next steps

- No action needed on #841 itself — correctly closed as duplicate.
- Triage/label #848 separately for the actionable fix (rough estimate: logging the drop is small; making `status-reconciler` diff against its own last-processed config to survive a dropped delta is moderate — touches the delta-consumption contract, needs a regression test for drop-then-recover).

## Open questions

- None outstanding on #841.
