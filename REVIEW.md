---
pr: kubernetes-sigs/prow#785
title: "spyglass/junit: deduplicate test detail row rendering"
head_sha: 88ef2524d460e6ae9ee700246be31c3c3a8735bf
base: main
reviewed_at: 2026-06-30T17:32:00Z
verdict: approve
---

## Summary

- Extracts a shared `{{define "test-detail-rows"}}` sub-template replacing four near-identical inline copies of expandable test result row markup (top-level failed, top-level flaky, group failed, group flaky).
- Unifies CSS class namespace: `failure-*`/`flaky-*`/`group-*` collapsed to `detail-*`.
- Updates TypeScript selector in `lens.ts` and test expectation in `lens_test.go` to match.
- Net -243/+85 lines, pure mechanical deduplication.

## Findings

No findings.

## Checked

- The unified template introduces `{{if eq $numTest 1}}` to flaky code paths that previously lacked it. Verified in `lens.go:303-321` that flaky classification requires >= 2 test runs (mixed pass/fail), so the single-run branch is unreachable for `.Flaky` data. No behavioral change.
- Grepped entire repo for old CSS class names (`failed-layout`, `flaky-layout`, `group-layout`, `failure-name`, `flaky-name`, `group-name`, `failure-text`, `flaky-text`, `group-text`). No dangling references remain.
- `lens_test.go` expectation updated from `group-layout` to `detail-layout`, consistent with the rename.
- TypeScript selector in `lens.ts` updated from `.failure-name,.group-name,.flaky-name` to `.detail-name`.
- Section-level identifiers (`failed-tbody`, `flaky-tbody`, `group-tbody`, `group-header`) intentionally unchanged — they serve a different purpose (section organization, not row rendering).

## Open questions

- None.
