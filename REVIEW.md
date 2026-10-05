---
pr: kubernetes-sigs/prow#781
title: "deck: filter tide history per-record instead of per-pool"
head_sha: de9a3cb9af63b4abf71f6956a13f07ffb443c16e
base: main
reviewed_at: 2026-07-27T00:21:32Z
verdict: approve
---

## What this PR does
- `filterHistory` used to union all `TenantIDs` across every record in a pool into one set (`recordIDs`), then kept or dropped the whole pool based on that union via `HasAll`.
- One record with a foreign tenant ID (e.g. a batch-merge leg) was enough to hide the entire pool's history, including records that should have been visible.
- Fix: `ta.filter` is now evaluated per-record, using only that record's own `TenantIDs`; the pool is kept (with only the matching records) if at least one record passes.
- `recordIDs` helper removed (no other callers); also removes a confusing no-op `Insert()` call with no arguments.
- Test updated: two existing expectations adjusted to reflect per-record semantics, plus one new case exercising mixed tenant IDs within a single pool.

## Findings

### [nit] pre-size filteredRecords slice
- where: `cmd/deck/tide.go:196`
- concern: `var filteredRecords []history.Record` grows via repeated `append` with no capacity hint, even though the upper bound (`len(records)`) is known. `make([]history.Record, 0, len(records))` avoids incremental reallocation for pools with many records. Negligible at typical history sizes.
- excerpt: |
    var filteredRecords []history.Record
    for _, record := range records {
        curIDs := sets.New[string](record.TenantIDs...)
        if match := ta.filter(orgRepoID, curIDs, needsHide); match {
            filteredRecords = append(filteredRecords, record)
        }
    }

### [nit] undocumented filtering-granularity asymmetry vs filterPools
- where: `cmd/deck/tide.go:189-204`
- concern: `filterPools` still filters at the aggregate/pool level (`pool.TenantIDs` as one set), while `filterHistory` now filters per-record. Defensible given the different data shapes (one pool vs. many discrete history records), but nothing notes this is intentional. A future maintainer editing one might assume the other behaves the same way.
- excerpt: |
    // filterHistory filters per-record (unlike filterPools, which filters
    // per-pool) because a pool's history can contain records with
    // heterogeneous tenant IDs, e.g. from batch merges.

## Checked
- `orgRepoID` injection into `curIDs` happens per-record now instead of once per pool, but `orgRepoID` is pool-level and identical across records in the loop, so this is behavior-neutral vs. the old per-pool computation.
- `recordIDs` has no remaining references after removal.
- `filterPools` (pool-level, not per-record) intentionally left untouched — pools don't have the same "some records legitimately belong to another tenant" problem.
- New test case ("per-record filtering keeps matching records when pool has mixed tenant IDs") covers default-only, mixed, private-only, and untagged records in one pool.
- Adjusted test expectations (`clustered-tenant/test:master` losing its untagged `MERGE_BATCH` record) correctly reflect the new semantics; traced through `exampleConfigNoDefaults` to confirm `orgRepoID` is empty for that repo.
- Per-record `sets.New[string](...)` allocation replaces the old once-per-pool allocation — extra allocations, but negligible for realistic history sizes.
- Deployment risk: display-only change in Deck, no config/API/persisted-state impact, fully backward-compatible for single-tenant deployments. Multi-tenant Deck operators will see more (previously hidden) history rows post-upgrade — intended, but a visible behavior change worth a release-note mention.

## Open questions
None.
