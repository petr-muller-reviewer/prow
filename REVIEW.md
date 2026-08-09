---
pr: kubernetes-sigs/prow#674
title: "tide: skip unmergeable PRs instead of retrying indefinitely"
head_sha: d6d08d0b2b40c9ef3fee39d3886a44be0c993f23
base: main
reviewed_at: 2026-08-09T01:38:33Z
verdict: request-changes
refresh_log:
  - from: 910f6c95a54b8370a167f1b8453ce1d53cb08ac5
    to: d6d08d0b2b40c9ef3fee39d3886a44be0c993f23
    summary: "Author addressed both should-fix findings: batch-merge path now filters/records exclusions, and TTL-based expiry was replaced with a sync-cycle counter aged unconditionally every Sync(), closing the unbounded-growth gap."
  - from: d6d08d0b2b40c9ef3fee39d3886a44be0c993f23
    to: d6d08d0b2b40c9ef3fee39d3886a44be0c993f23
    summary: "No new commits. A second-pass code review (with a working Go toolchain) found two should-fix gaps the prior pass missed: the serial-trigger gate still tests unfiltered `successes`, and the pre-existing-batch merge path never consults the exclusion set. Verdict downgraded approve -> request-changes; the prior 'Resolved: batch-merge path' entry was only partially correct."
---

## What this PR does
- Fixes #673: `takeAction()` deterministically re-picks the lowest-numbered PR from `successes` every sync cycle; if that PR fails to merge with `UnmergablePRError` (e.g. branch protection needs more approvals), it blocks the whole queue forever.
- Adds `syncController.mergeExclusions map[string]int` keyed by `org/repo#number@sha`, with `excludeFromMerge`/`isExcludedFromMerge`/`filterExcludedFromMerge` (mutex-guarded).
- `isUnmergableError()` unwraps `*mergeFailure` → `mergeErr.err` → `fmt.Errorf`-wrapped `github.UnmergablePRError` via `errors.As`.
- `takeAction()` filters `successes` through `filterExcludedFromMerge` before `pickHighestPriorityPR`, and calls `excludeFromMerge` when `mergePRs` returns an unmergeable error; `pickBatch` filters candidates too, and `excludeUnmergablePRsFromBatch` records per-PR exclusions after a failed batch merge.
- Entries age via `ageMergeExclusions()`, called once per `Sync()`; `mergeExclusionSyncs = 3`. SHA-scoped key means a new push auto-clears the exclusion without waiting for it to age out.

Since previous review: no new commits (`head_sha` unchanged). A second review pass was run with a working Go toolchain (`go test ./pkg/tide/...` and `go vet ./pkg/tide/` both pass — the first pass could not execute either).
- Two new `should-fix` findings, both in `takeAction`: the serial-trigger gate and the pre-existing-batch merge path each still operate on unfiltered data, so an excluded PR can still block other PRs.
- The prior "Resolved — batch-merge path bypassed the exclusion filter" entry is downgraded to partially resolved: `pickBatch` selection and post-failure exclusion recording were fixed, but merging an already-successful batch was not.
- Verdict downgraded from `approve` to `request-changes`.

## Findings

### [should-fix] Serial-trigger gate still tests unfiltered `successes`, so an excluded PR keeps blocking other PRs' tests
- where: `pkg/tide/tide.go:1741`
- concern: the direct-merge branch filters into `eligible`, but the fall-through serial-trigger condition still reads `len(successes) == 0` using the original slice. A PR that is excluded from merging therefore still counts as a "success" that suppresses serial test triggers. Concretely: pool has PR #1 (green, permanently rejected by branch protection) and PR #2 (missing required presubmits). #1 is excluded, `eligible` is empty, so we fall through — but `len(successes) == 1`, so #2's presubmits are never triggered. `pickBatch` doesn't rescue it either, since #2 isn't retest-eligible. This is the same class of head-of-line blocking the PR set out to fix, moved one step downstream. Gate on `eligible` instead.
- excerpt: |
    eligible := c.filterExcludedFromMerge(sp.log, sp.org, sp.repo, successes)
    ...
    if len(missings) > 0 && len(pendings) == 0 && len(successes) == 0 {

### [should-fix] Pre-existing batch merges bypass the exclusion set entirely
- where: `pkg/tide/tide.go:1695`, doc comment at `pkg/tide/tide.go:615-621`
- concern: `takeAction` merges `batchMerges` unconditionally — only `pickBatch` (new batch selection) and the single-merge path run through `filterExcludedFromMerge`. `batchMerges` comes from `accumulateBatch`, which returns any still-valid successful batch ProwJob, so a batch containing an excluded PR is retried whole every cycle. The practical exposure is narrower than it looks: if the excluded PR's batch-mates merge successfully they leave the pool, invalidating the batch refs, so sustained retry requires *every* other batch member to also fail (transiently) on the same cycle. What is unambiguously wrong is the doc comment on `excludeUnmergablePRsFromBatch`, which asserts this case is handled ("whether the batch is retried whole or the PR resurfaces in a later batch"). Either filter `batchMerges` (skipping the batch if any member is excluded) or correct the comment.
- excerpt: |
    // Merge the batch!
    if len(batchMerges) > 0 {
        merged, err = c.provider.mergePRs(sp, batchMerges, c.statusUpdate.dontUpdateStatus)

### [nit] `ageMergeExclusions()` ages on syncs that never attempt a merge
- where: `pkg/tide/tide.go:670`
- concern: it is the first statement of `Sync()`, before `provider.Query()`, `blockers()` and `dividePool()`. It also ages during syncs where the subpool is `PoolBlocked` and `takeAction` is never reached. Three consecutive syncs that fail on `failed getting blockers` will expire an exclusion having granted Tide zero actual opportunities to advance to the next PR. Aging after subpools are synced — or only counting cycles in which `takeAction` ran — would match the documented intent that the window "behaves the same regardless of how frequently Tide is configured to sync". Low impact: worst case the unmergeable PR is retried one cycle early.

### [nit] `pickBatch` log message misattributes an exclusion to failing tests
- where: `pkg/tide/tide.go:1358-1363`
- concern: `filterExcludedFromMerge` is applied to `candidates` immediately before the `len(candidates) == 0` check, but the message still reads "None of the prs in the subpool was passing tests, no batch will be created". When all candidates are merge-excluded (they *did* pass tests), an operator debugging "why is no batch being created" is pointed at CI rather than at the exclusion. Distinguish the two causes, or move the filter after the check.
- excerpt: |
    candidates = c.filterExcludedFromMerge(sp.log, sp.org, sp.repo, candidates)
    if len(candidates) == 0 {
        log.Debug("None of the prs in the subpool was passing tests, no batch will be created")

### [nit] Per-cycle skip is logged at Debug while the original exclusion is Warning
- where: `pkg/tide/tide.go:588` vs `1673`
- concern: the initial "excluded" event logs at `Warning`, but every subsequent cycle's skip logs only at `Debug` (usually off in production). No history record is written for the skips either, and the PR's Tide status context still reports it as in the merge pool — so the author sees a green, "in merge pool" PR that Tide silently declines to touch. Promoting the skip to `Info` (volume is bounded by pool size) would make this diagnosable.
- excerpt: |
    log.WithFields(pr.logFields()).Debug("Skipping PR temporarily excluded from merging due to recent merge failure.")

### [nit] `excludeFromMerge` will panic on a nil `mergeExclusions` map
- where: `pkg/tide/tide.go:104-105`, `538-552`
- concern: `mergeExclusions` is only initialized in `newSyncController`, but `tide_test.go` constructs `&syncController{...}` literals in ~15 places, and the two new merge-failure tests had to remember `mergeExclusions: make(map[string]int)`. No live bug today — every production path goes through the constructor and no existing test hits the write path — but a lazy init inside `excludeFromMerge` removes a sharp edge for the next test or constructor added. `isExcludedFromMerge`'s read is nil-safe.
- excerpt: |
    mergeExclusions   map[string]int
    mergeExclusionsMu sync.Mutex

### [nit] `isUnmergableError` duplicates knowledge of `mergeFailure`'s internal shape
- where: `pkg/tide/tide.go:599-613`
- concern: iterates `mf.errs` and recurses, duplicating structural knowledge that also lives in `mergeFailure.historyMessage()`/`operatorError()` and now in `excludeUnmergablePRsFromBatch()`. A `mergeFailure.underlyingErrors() []error` helper would consolidate. Not urgent.

### [question] Should `mergeExclusionSyncs` (3 cycles, hardcoded) be configurable?
- where: `pkg/tide/tide.go:519`
- concern: for large multi-tenant installations, a branch-protection misconfiguration can take a human longer than three sync cycles to notice and fix, causing the PR to flap in and out of the queue with repeated Warning logs. A `Tide` config knob and/or a `tide_merge_exclusions_active` metric would give operators visibility without a code change. The switch from wall-clock TTL to cycle count makes the window's real-world duration depend on the configured sync period, worth documenting if not made configurable.

## Resolved

### [was should-fix] Exclusion map entries for abandoned PRs were never cleaned up
- Fixed in `d6d08d0b2` by switching from lazy wall-clock expiry to a sync-cycle counter: `ageMergeExclusions()` is called once per `Sync()`, decrementing (and deleting at zero) every entry regardless of whether it is looked up again. A PR that never resurfaces in `successes` (closed, abandoned, loses labels) is cleaned up within `mergeExclusionSyncs` cycles. `excludeFromMerge` additionally drops the PR's prior-SHA entries on re-exclusion. Covered by `TestMergeExclusionAging`.

### [partially resolved, was should-fix] Batch-merge path bypassed the exclusion filter
- Fixed in `d6d08d0b2` for two of three sub-paths: `pickBatch` now runs candidates through `filterExcludedFromMerge()` before batch selection, and `excludeUnmergablePRsFromBatch()` inspects a failed batch merge's per-PR errors and excludes only the PR(s) that returned `UnmergablePRError`, leaving batch-mates alone. Covered by `TestTakeActionExcludesPRThatFailsInBatch` and `TestPickBatchExcludesMergeExcludedPR`.
- Not fixed: merging an already-successful batch (`batchMerges` in `takeAction`) — see the open `should-fix` above. The prior review recorded this finding as fully resolved; that was wrong.

## Checked
- `go test ./pkg/tide/...` and `go vet ./pkg/tide/` both pass (the previous review pass had no Go toolchain and relied on static tracing only).
- Full error-unwrap chain traced end-to-end: `tryMerge()` wraps `github.UnmergablePRError` as `fmt.Errorf("PR is unmergable...: %w", err)` → `GitHubProvider.mergePRs()` wraps into `&mergeFailure{errs: []mergeErr{...}}` → `isUnmergableError()` type-asserts `*mergeFailure`, recurses into `me.err`, falls back to `errors.As`. Matches the actual runtime error shape.
- `errors.As` works against `github.UnmergablePRError` (value receiver on `Error()`), and correctly does *not* match the distinct `UnmergablePRBaseChangedError`, `UnauthorizedToPushError` or `MergeCommitsForbiddenError` types — the predicate is not overly broad.
- Key scheme is collision-safe: `o/r#1@` cannot prefix-match `o/r#12@`, so the prior-SHA cleanup loop in `excludeFromMerge` won't delete unrelated PRs' entries.
- Aging arithmetic matches the constant: a PR excluded during sync N is skipped for N+1..N+3 and retried at N+4. The `remaining > 0` check in `isExcludedFromMerge` is dead (zero-valued entries are deleted) but harmless.
- SHA-keyed invalidation genuinely works: `filterExcludedFromMerge` checks the current `HeadRefOID`, so a force-push clears the exclusion immediately.
- `mergeExclusionsMu` is the correct primitive and correctly scoped: `syncSubpool` runs concurrently across org/repo/branch subpools, the map is genuinely shared, locking is flat (no nested acquisition between `filterExcludedFromMerge` and `isExcludedFromMerge`), and `syncController` is only used through a pointer so the added mutex introduces no copylock issue.
- `takeAction` only calls `mergePRs` with a single-element slice on the non-batch path, so `mergeFailure.errs` has exactly one entry there — no ambiguity about which PR triggered the exclusion.
- New tests (`TestTakeActionSkipsExcludedPRs`, `TestIsUnmergableError`, `TestMergeExclusionAging`, `TestExcludeFromMergeReplacesPriorSHA`, `TestTakeActionExcludesPRThatFailsInBatch`, `TestPickBatchExcludesMergeExcludedPR`) are structurally consistent with existing helpers; the two `takeAction` ones are genuine end-to-end regression tests, not unit tests of map internals.
- Feature is GitHub-only (Gerrit provider untouched), consistent with the PR's stated scope. No config/API/schema changes; all state is in-memory and lost on restart, so the change is drop-in and safely revertible.
- A claim surfaced during the first review that this PR also changes `mergeFailure`/`historyMessage`/`operatorError` redaction behavior, an `interface{}`→`any` signature, and removes two `Debug` log lines was checked against `gh pr diff 674` and found to be **false** — that code is pre-existing and untouched. Disregarded.

## Open questions
- The serial-trigger gate (`len(successes) == 0`) looks like it should be `len(eligible) == 0` — was gating on the unfiltered slice deliberate, or an oversight? As written, an unmergeable PR still starves other PRs of serial tests, which is the failure mode #673 describes.
- Should `batchMerges` be filtered through `filterExcludedFromMerge` as well, or is the "batch retried whole" case considered unreachable in practice? If the latter, the doc comment on `excludeUnmergablePRsFromBatch` should be narrowed, since it currently claims that case is covered.
- Should `mergeExclusionSyncs` be configurable, and is a metric for active exclusions worth adding now vs. as a follow-up?
