---
pr: kubernetes-sigs/prow#870
title: "statusreconciler: never drop config deltas (fix #848)"
head_sha: f546617672b2e50bea2d8626706f407a53f4d715
base: main
reviewed_at: 2026-10-13T10:33:08Z
verdict: approve
refresh_log:
  - old_sha: ed41879ed05957591fd2e5a7bca5d2056c40c6ff
    new_sha: 4ce803f6e239c73555a74ba054db7587357599f5
    summary: "Rebased the existing change onto current main, simplified delta delivery, and added concurrent and independent-subscriber coverage; resolved the deliverDelta readability nit."
  - old_sha: 4ce803f6e239c73555a74ba054db7587357599f5
    new_sha: f546617672b2e50bea2d8626706f407a53f4d715
    summary: "Rebased to resolve conflicts, migrated the newly added prow-controller-manager caller after CI exposed a compile failure, and recorded Prucek's approval and the merge."
gate:
  decision: merge
  gated_at: 2026-10-07T17:13:20Z
  gated_head_sha: 4ce803f6e239c73555a74ba054db7587357599f5
  reviewed_head_sha: 4ce803f6e239c73555a74ba054db7587357599f5
---

## Gate

**Verdict: merge.**

The maintainer accepted the `pkg/config` API change: internal consumers are ported and external migration is trivial. This gate was recorded at `4ce803f6e239c73555a74ba054db7587357599f5`. After CI exposed a stale caller, the PR migrated it at `f546617672b2e50bea2d8626706f407a53f4d715`; that head was approved by Prucek and merged as `b4d923a1a60cedd794f9461c529df11ebd8e1abc` on October 13.

### Gating list

- None. @Prucek's concern about downstream compatibility is dispositioned by the maintainer's acceptance of the API break and confirmation that migration is straightforward.

### Independent merge risk

- **Backward-incompatible Go API:** downstream modules importing `sigs.k8s.io/prow/pkg/config` and calling `Agent.Subscribe` need a source update; existing built binaries are unaffected. This is not opt-in and has no compatibility shim or separate migration note. The maintainer accepts this break and says migration is trivial.
- **Changed event behavior:** the single-slot buffer coalesces unread deltas, so slow subscribers receive the earliest `Before` and latest `After` rather than every intermediate config transition. Internal consumers are ported; external consumers relying on each delta as a distinct event need to adapt. The coalescing behavior is described in code comments.

## What this PR does

- Fixes #848: `config.Agent.Set` used to deliver each delta via a per-send goroutine with a
  1-minute timeout, silently dropping the delta if the subscriber was still busy — the
  status-reconciler relies on `Before` chaining to the previous `After`, so a dropped delta
  permanently hid config changes.
- Makes delivery lossless at the source: `Subscribe()` now creates and owns a single-slot
  buffered channel, returning it as a receive-only `DeltaChan` (`<-chan Delta`), instead of
  taking a caller-supplied channel.
- `Set` calls the new `deliverDelta` helper, which coalesces into the single slot instead of
  blocking or dropping: if a pending `{Before, _}` hasn't been drained, it's taken back and
  replaced with `{Before, newAfter}`, widening the delta to the transition the subscriber
  still owes.
- Removes the per-send goroutine and one-minute timer entirely; fixes the bug for every
  subscriber, not just status-reconciler.
- Updates all four call sites (`moonraker`, `pubsub/subscriber`, `statusreconciler`, and
  `prow-controller-manager`) to the
  new ownership model — they no longer allocate their own channel and just take back what
  `Subscribe()` returns.
- Adds tests in `pkg/config/agent_test.go` covering coalescing under a busy subscriber,
  chained delivery, concurrent receives, and independent subscriber coalescing.

Since previous review:

- The PR was rebased and force-pushed to `0b67715d2d0dc0287dc0395dfea2a197a74235a0` on
  October 1. On October 7, Petr Muller added `4ce803f6e239c73555a74ba054db7587357599f5`
  (`pkg/config/agent.go`, `pkg/config/agent_test.go`; 106 additions, 15 deletions), replacing
  the coalescing loop with a drain-and-retry sequence and adding concurrent-receive and
  independent-subscriber tests. This resolves the `deliverDelta` readability nit.
- On October 1, `kubernetes-prow[bot]` reported new changes and removed the `lgtm` and
  `do-not-merge/invalid-commit-message` labels. On October 7, Petr Muller commented `/hold
  cancel`; the bot removed `do-not-merge/hold`. No inline review comments or submitted reviews
  were added.

Since previous review (October 12–13):

- Petr Muller rebased and force-pushed the PR to resolve conflicts. The old head is no longer
  reachable; GitHub's current PR diff is six paths (+199/−37), including
  `cmd/prow-controller-manager/main.go` (+1/−2). The raw tip-to-tip diff spans 44 files due to
  base advancement; the PR's feature scope remains the config-delta change.
- CI on the intermediate head `1796e845aa24e5702d2a0462fd6a0769c1524d95` failed compilation
  because `cmd/prow-controller-manager/main.go:135` still passed a channel to the new
  no-argument `Subscribe()`. Head `f546617672b2e50bea2d8626706f407a53f4d715` fixes it with
  `configChanges := configAgent.Subscribe()`. Prucek approved that head at 08:31:14 UTC on
  October 13, and the PR merged as `b4d923a1a60cedd794f9461c529df11ebd8e1abc` at 08:32:21 UTC.

## Resolved

### [blocking] `prow-controller-manager` used the old `Subscribe` signature
- where: `cmd/prow-controller-manager/main.go:135` on intermediate head `1796e845aa24e5702d2a0462fd6a0769c1524d95`
- concern: the call passed a caller-allocated channel to `Subscribe`, but the PR changed the
  method to take no arguments and return a receive-only channel. CI reported a compile failure
  in the Prow unit-test, integration, and image-build jobs.
- excerpt: |
    configChanges := make(chan config.Delta, 1)
    configAgent.Subscribe(configChanges)
- resolution: head `f546617672b2e50bea2d8626706f407a53f4d715` changes the call to
  `configChanges := configAgent.Subscribe()`, matching the new API and the watcher's
  receive-only channel field.

### [nit] `deliverDelta`'s double-select could be linearized
- where: `pkg/config/agent.go:434-451`
- concern: given the documented single-producer/single-consumer invariant (`Set` is
  serialized by `ca.mut`, and nothing else writes to `sub`), the loop always resolves in at
  most two iterations — try-send, then (if full) drain-and-merge, then guaranteed send. It
  could be written as a straight-line sequence instead of a `for { select { ... select ... } }`,
  making the "no livelock" guarantee visible from the control flow rather than only from the
  comment. Purely a readability suggestion, not a correctness issue.
- excerpt: |
    func deliverDelta(sub chan Delta, delta Delta) {
        for {
            select {
            case sub <- delta:
                return
            default:
                select {
                case pending := <-sub:
                    delta = Delta{Before: pending.Before, After: delta.After}
                default:
                }
            }
        }
    }
- resolution: commit `4ce803f6e239c73555a74ba054db7587357599f5` replaces the retry loop with
  two straight-line selects and a final send, making the bounded delivery path explicit.

## Checked

- Channel-ownership change (`Subscribe` returning `DeltaChan` instead of taking a
  caller-supplied channel) — traced all three call sites (`moonraker.go`,
  `pubsub/subscriber/server.go`, `statusreconciler/status.go`); all updated consistently and
  compile against the new signature.
- Delta-chaining correctness: since `Before` on the merged delta is the original pending
  `Before` and deltas always chain (each `Before` equals the previous `After`), coalescing
  preserves the invariant subscribers rely on.
- Removed the old goroutine/timeout drop path does not regress moonraker or pubsub — both
  already track their own last-seen config independently and don't depend on the drop
  behavior.
- New tests (`TestSetCoalescesForBusySubscriber`, `TestSetDeliversChainedDeltasWhenDrained`)
  exercise both the coalescing path and the keep-up path correctly.
- No other subscribers or `DeltaChan`/`Subscribe` usages left on the old API in the repo.

## Open questions

- None — the change is small, well-contained, and the reasoning is spelled out clearly in
  the `deliverDelta` doc comment.

## Followups

Recorded by `/review:followup` on 2026-10-25T13:54:44Z for merged PR #870
(`b4d923a1a60cedd794f9461c529df11ebd8e1abc`). Two of six candidates accepted:

### [tests] Probe coalescing edge cases and `Load()` chaining
- category: tests
- necessity: could
- mode: probe
- where: `pkg/config/agent.go:454-470`, `pkg/statusreconciler/status.go:57-70`
- why followup: merged tests cover coalescing, chained delivery, concurrent receive, and
  independent subscribers, but the `deliverDelta` branch taken when the subscriber drains
  the slot between the failed send and the drain attempt (`pkg/config/agent.go:466-467`)
  has no deterministic exercise, and nothing stresses `statusController.Load()` when config
  is rewritten repeatedly during startup. A probe is cheap and its only deliverable is a
  bug fix.
- handoff prompt:

```text
In kubernetes-sigs/prow (default branch `main`), following PR #870 "statusreconciler: never
drop config deltas" (merged as b4d923a1a60cedd794f9461c529df11ebd8e1abc), run a throwaway
test probe over the new config-delta coalescing code and hunt for real bugs.

Context: `pkg/config/agent.go` implements `Agent.Subscribe() DeltaChan` (single-slot buffer)
and `deliverDelta` (try-send; if full, drain the pending delta and merge it, preserving its
`Before`; then a final send). `Set` holds `ca.mut` while delivering, and the receive-only
`DeltaChan` alias prevents subscribers from writing. Existing tests live in
`pkg/config/agent_test.go`; `pkg/statusreconciler/status_test.go` covers
`statusController.Load`.

Task: write throwaway tests (uncommitted, in the normal test locations) exercising:
1. The no-pending-delta branch of `deliverDelta` (subscriber drained between the failed send
   attempt and the drain attempt). Drive it with a start barrier and many iterations under
   `-race`, asserting the consumer sees a continuous chain — every `Before` equals the
   previous `After`, with no delta lost or duplicated.
2. A subscriber that never drains across many `Set` calls and then drains once. Assert
   `Set` never blocks and the reader receives exactly one delta, `{first Before, last
   After}`.
3. `statusController.Load()` while the config files are rewritten repeatedly, so several
   `Set` calls happen before the first receive. Assert the returned delta's `Before` is the
   persisted state and `After` is the final config.
Run `go test -race -count=20 ./pkg/config/... ./pkg/statusreconciler/...` and classify each
failure as a real implementation bug or a wrong test expectation, with evidence.

Acceptance criteria:
- All three behaviours are exercised; every failure is classified with the failing case and
  the observed vs. expected delta.
- If a real bug is found: apply the minimal fix only, and report the failing case so it can
  be kept as a regression test.
- If no real bug is found: no files changed; delete the probe tests.

Scope guard: the deliverable is a bug fix, never the probe tests; do not refactor
`deliverDelta` or `Subscribe` for style; do not touch unrelated tests or packages; do not
change coalescing semantics unless a concrete correctness failure proves them wrong.
```

### [tech-debt] Dedupe the two config-watch loops
- category: tech-debt
- necessity: could
- mode: deliver
- where: `pkg/moonraker/moonraker.go:100-140`, `pkg/pubsub/subscriber/server.go:151-200`
- why followup: #870 touched both call sites and left two structurally identical loops —
  subscribe, snapshot the watched sections, select on context/channel, `reflect.DeepEqual`,
  restart the worker. The only differences are which sections are watched and moonraker's
  extra `SetWithoutBroadcast` step. The duplication predates the PR; the shared `DeltaChan`
  API is what makes extracting it natural now.
- handoff prompt:

```text
In kubernetes-sigs/prow (default branch `main`), following PR #870 "statusreconciler: never
drop config deltas" (merged as b4d923a1a60cedd794f9461c529df11ebd8e1abc), remove the
duplicated config-watch loops in moonraker and the pubsub subscriber by extracting a shared
helper.

Context: `config.Agent.Subscribe()` now returns an agent-owned, receive-only
`config.DeltaChan` whose single-slot buffer coalesces undrained updates. Both
`(*Moonraker).RunConfigWatcher` (`pkg/moonraker/moonraker.go:100-140`) and
`(*PullServer).Run` (`pkg/pubsub/subscriber/server.go:151-200`) follow the same shape:
subscribe; snapshot the sections they care about (`configSectionsToWatch` embeds
`config.InRepoConfig`; `configToWatch` embeds `config.PubSubTriggers` and
`config.PubsubSubscriptions`); loop on `ctx.Done()`, the delta channel, and — for the pull
server — a derived context; rebuild the snapshot on each delta and restart the worker only
when `reflect.DeepEqual` reports a change. Moonraker additionally calls
`mr.ConfigAgent.SetWithoutBroadcast(&event.After)` before recording its new snapshot.

Task:
1. Add a reusable helper in `pkg/config` that encapsulates subscribe -> snapshot -> select
   -> compare -> notify. Because the snapshots contain maps, compare with
   `reflect.DeepEqual` on `any` (or take a caller-supplied equality func) rather than a
   `comparable` constraint. It must:
   - return on caller context cancellation,
   - watch the latest config only (the channel already coalesces; do not try to replay
     intermediate transitions),
   - accept an optional extra done-channel for the pull server's derived context,
   - invoke a caller-provided post-change hook before recording the new snapshot, so
     moonraker can keep its `SetWithoutBroadcast` step,
   - propagate hook errors to the caller.
2. Migrate both call sites to the helper with no behaviour change, and delete the duplicated
   loops. Keep log lines equivalent (`Received new config`, `New config found, ...`).

Acceptance criteria:
- `go build ./pkg/...` and `go vet ./pkg/config/... ./pkg/moonraker/... ./pkg/pubsub/...`
  pass; `go test ./pkg/config/... ./pkg/moonraker/... ./pkg/pubsub/...` passes.
- Unit tests for the helper: an unchanged-snapshot delta does not fire the hook; a changed
  snapshot does; caller context cancellation returns; the extra done-channel returns; hook
  errors propagate.
- Moonraker still re-sets the agent config without broadcasting when `InRepoConfig`
  changes and still ignores changes to unrelated sections.
- Neither old loop remains; no duplicated subscribe/compare code is left in either package.

Scope guard: do not change `Agent.Subscribe`, `Set`, `SetWithoutBroadcast`, or the
coalescing semantics; do not change which sections each component watches or when it
restarts; do not touch other config consumers (status-reconciler, prow-controller-manager);
keep the helper's exported surface minimal since `pkg/config` is imported widely; do not
reformat unrelated code.
```
