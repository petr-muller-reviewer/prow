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
