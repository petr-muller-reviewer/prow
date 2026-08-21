---
pr: kubernetes-sigs/prow#864
title: "checkconfig: don't require tide to forbid needs-rebase"
head_sha: 2359ee04392fb8795e1be0457478f2172a639b2c
base: main
reviewed_at: 2026-08-20T19:23:31Z
verdict: approve
---

## Summary

- Single commit, touches `cmd/checkconfig/main.go` and `cmd/checkconfig/main_test.go`.
- Previously, `checkconfig` required that any label-based plugin (e.g. `needs-rebase`) both be forbidden as a merge label by tide's `MissingLabels`/query config *and* something else in the opposite direction; this PR drops the requirement that tide's query forbid `needs-rebase` specifically.
- Refactored the check into a closure/extracted helper (`populateConfig`, `ensureLabelPluginEnabled`) while preserving the two-directional check for all other plugin/label pairs.
- Added a focused unit test `TestEnsureLabelPluginEnabled` with three cases covering the new semantics.

## Findings

None.

## Checked
- Verified `mergeChecker.isAllowedToMerge` (pkg/tide/github.go:608) blocks merging on `MergeableStateConflicting` independently of the tide query's `MissingLabels` config — this is the load-bearing fact justifying dropping the "notRequired" direction for needs-rebase specifically.
- Confirmed the refactor preserves existing behavior for all other plugin/label pairs (only the needs-rebase direction changed).
- Searched for other callers/consumers of the removed check elsewhere in the repo — none found.
- New unit test `TestEnsureLabelPluginEnabled` cases match the intended semantics.
- No repo-level or directory-level CLAUDE.md applies to `cmd/checkconfig/`.

## Open questions
None.
