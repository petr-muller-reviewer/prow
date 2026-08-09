---
pr: kubernetes-sigs/prow#674
title: "tide: skip unmergeable PRs instead of retrying indefinitely"
head_sha: d6d08d0b2b40c9ef3fee39d3886a44be0c993f23
base: main
reviewed_at: 2026-08-04T12:59:26Z
verdict: approve
refresh_log:
  - from: 910f6c95a54b8370a167f1b8453ce1d53cb08ac5
    to: d6d08d0b2b40c9ef3fee39d3886a44be0c993f23
    summary: "Author addressed both should-fix findings: batch-merge path now filters/records exclusions, and TTL-based expiry was replaced with a sync-cycle counter aged unconditionally every Sync(), closing the unbounded-growth gap."
---

## What this PR does
- Fixes #673: `takeAction()` deterministically re-picks the lowest-numbered PR from `successes` every sync cycle; if that PR fails to merge with `UnmergablePRError` (e.g. branch protection needs more approvals), it blocks the whole queue forever.
- Adds `syncController.mergeExclusions map[string]int` keyed by `org/repo#number@sha`, with `excludeFromMerge`/`isExcludedFromMerge` (mutex-guarded).
- `isUnmergableError()` unwraps `*mergeFailure` → `mergeErr.err` → `fmt.Errorf`-wrapped `github.UnmergablePRError` via `errors.As`.
- `takeAction()` filters `successes` through `isExcludedFromMerge` before `pickHighestPriorityPR`, and calls `excludeFromMerge` when `mergePRs` returns an unmergeable error.
- SHA-scoped key means a new push (new head SHA) auto-clears the exclusion without waiting for it to age out.

Since previous review: the author pushed a second commit (`d6d08d0b2`) addressing both `should-fix` findings:
- Extracted the exclusion filter into `filterExcludedFromMerge()` and applied it to the batch path too (`pickBatch`, before candidates are considered), plus added `excludeUnmergablePRsFromBatch()` which inspects a failed batch merge's per-PR errors and excludes only the individual PR(s) that came back `UnmergablePRError`, leaving batch-mates alone.
- Replaced the wall-clock TTL (`map[string]time.Time`, 5-minute `mergeExclusionTTL`) with a sync-cycle counter (`map[string]int`, `mergeExclusionSyncs = 3`) aged by a new `ageMergeExclusions()`, called unconditionally once per `Sync()` before subpools are processed. Since every entry is decremented (and deleted at zero) on every cycle regardless of whether it's looked up, this also closes the unbounded-growth gap for abandoned/closed PRs that never resurface in `successes`.
- Added `TestExcludeFromMergeReplacesPriorSHA`, `TestTakeActionExcludesPRThatFailsInBatch` (batch mixed pass/fail regression test), and `TestPickBatchExcludesMergeExcludedPR`; renamed `TestMergeExclusionExpiry` → `TestMergeExclusionAging` to match the new mechanism.

## Findings

### [nit] `excludeFromMerge` will panic on a nil `mergeExclusions` map
- where: `pkg/tide/tide.go:99-105`, `536-550`
- concern: `mergeExclusions` is only initialized in `newSyncController`. Several existing tests build `&syncController{...}` literals directly without setting it; none currently call `takeAction` with an unmergeable single-PR success, so there's no live bug today, but `excludeFromMerge`'s map write will panic on a nil map the first time a future test/path does. `isExcludedFromMerge`'s read is nil-safe. A defensive lazy-init or an invariant comment removes the footgun.
- excerpt: |
    mergeExclusions   map[string]int
    mergeExclusionsMu sync.Mutex

### [nit] Per-cycle skip is logged at Debug while the original exclusion is Warning
- where: `pkg/tide/tide.go:1649` vs `1663`
- concern: the initial "excluded" event logs at `Warning`, but every subsequent cycle's "still skipping" logs only at `Debug` (usually off in production). An operator investigating a stuck PR days after the original Warning scrolled out of retention has no way to see why it's still being skipped. Promoting the skip log to `Info` would help incident debugging without adding much volume (bounded by pool size).
- excerpt: |
    sp.log.WithFields(pr.logFields()).Debug("Skipping PR temporarily excluded from merging due to recent merge failure.")

### [nit] `isUnmergableError` duplicates knowledge of `mergeFailure`'s internal shape
- where: `pkg/tide/tide.go:570-586`
- concern: iterates `mf.errs` and recurses, duplicating structural knowledge that also lives in `mergeFailure.historyMessage()`/`operatorError()`. If `mergeFailure`'s shape changes, this is a third place that needs to stay in sync. A `mergeFailure.underlyingErrors() []error` helper would reduce duplication. Not urgent.

### [question] Should `mergeExclusionSyncs` (3 cycles, hardcoded) be configurable?
- concern: raised by the deployment-risk pass, originally against the 5-minute TTL — the concern carries over to the new sync-cycle counter. For large multi-tenant installations, a branch-protection misconfiguration can take a human longer than a few sync cycles to notice/fix, causing the PR to flap in and out of the queue with repeated Warning logs. A `Tide` config knob and/or a `tide_merge_exclusions_active` metric would give operators visibility without requiring a code change to tune. The switch from wall-clock to cycle-count (per the author's stated rationale in the code comment) makes the exclusion window's real-world duration depend on the configured sync period, which is worth calling out in docs if not made configurable.

## Resolved

### [was should-fix] Batch-merge path bypassed the exclusion filter entirely
- Fixed in `d6d08d0b2`: `takeAction`'s batch-merge branch now calls the new `excludeUnmergablePRsFromBatch()` on failure, which inspects the batch's per-PR errors and excludes only the PR(s) that failed with `UnmergablePRError` (batch-mates that merged fine are untouched). `pickBatch` also now runs candidates through `filterExcludedFromMerge()` before batch selection, so a recently-excluded PR can't immediately re-enter a new batch either. Covered by `TestTakeActionExcludesPRThatFailsInBatch` and `TestPickBatchExcludesMergeExcludedPR`.

### [was should-fix] Exclusion map entries for abandoned PRs were never cleaned up
- Fixed in `d6d08d0b2`, as a side effect of switching from lazy wall-clock expiry to a sync-cycle counter: `ageMergeExclusions()` is now called unconditionally once per `Sync()`, decrementing (and deleting at zero) every entry in the map regardless of whether it's looked up again. A PR that never resurfaces in `successes` (closed, abandoned, loses labels) is still cleaned up within `mergeExclusionSyncs` cycles. Covered by `TestMergeExclusionAging` (renamed from `TestMergeExclusionExpiry`).

## Checked
- Full error-unwrap chain traced end-to-end: `tryMerge()` wraps `github.UnmergablePRError` as `fmt.Errorf("PR is unmergable...: %w", err)` → `GitHubProvider.mergePRs()` wraps into `&mergeFailure{errs: []mergeErr{...}}` → `isUnmergableError()` type-asserts `*mergeFailure`, recurses into `me.err`, falls back to `errors.As`. Matches the actual runtime error shape, not a hypothetical one.
- `takeAction` only calls `mergePRs` with a single-element slice on the non-batch path, so `mergeFailure.errs` always has exactly one entry there — no ambiguity about which PR triggered the exclusion.
- Exclusion set/check both key off `pr.HeadRefOID` consistently.
- `mergeExclusionsMu` is the correct primitive: `syncSubpool` runs concurrently across org/repo/branch subpools via goroutines, so the shared map genuinely needs locking; both accessors take/release it around the full read-modify-write sequence.
- New tests (`TestTakeActionSkipsExcludedPRs`, `TestIsUnmergableError`, `TestMergeExclusionAging`, `TestExcludeFromMergeReplacesPriorSHA`, `TestTakeActionExcludesPRThatFailsInBatch`, `TestPickBatchExcludesMergeExcludedPR`) are structurally consistent with existing test helpers (`newSyncController`, `newGitHubProvider`, `newFakeManager`, `fgc.mergeErrs`); argument counts/types match current signatures. `TestTakeActionSkipsExcludedPRs` and `TestTakeActionExcludesPRThatFailsInBatch` are genuine end-to-end regression tests (single-PR and batch paths respectively), not unit tests of map internals.
- `isUnmergableError` correctly returns false for `UnauthorizedToPushError` and generic errors — not overly broad.
- Feature is GitHub-only (Gerrit provider untouched); consistent with the PR's stated scope.
- No config/API/schema changes; drop-in, safely revertible (all state is in-memory, lost on restart).
- A claim surfaced mid-review that this PR also changes `mergeFailure`/`historyMessage`/`operatorError` redaction behavior, an `interface{}`→`any` signature, and removes two `Debug` log lines was checked against `gh pr diff 674` directly and found to be **false** — that code is pre-existing and untouched by this PR. Disregarded.
- Could not run `go test ./pkg/tide/...` in this environment (no `go` binary available) — relied on static tracing of signatures and control flow instead of executing the suite.

## Open questions
- Should `mergeExclusionSyncs` be configurable, and would a metric for active exclusions be worth adding now vs. as a follow-up?

Resolved since previous review: the batch-merge exclusion gap and the unbounded exclusion-map growth question (see Resolved section above).
