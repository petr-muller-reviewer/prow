---
pr: kubernetes-sigs/prow#852
title: "chore: label dependabot PRs with area/dependency and ok-to-test"
head_sha: e8af58751a2c75e9ae15743e1d1bba9ae1364a09
base: main
reviewed_at: 2026-08-17T13:01:00Z
verdict: approve
---

## Summary

Single-commit change to `.github/dependabot.yaml`: adds a `labels:` key (`area/dependency`, `ok-to-test`) to the sole `gomod` update entry, so Dependabot PRs get labeled and skip the manual `/ok-to-test` comment.

## Findings

(none)

## Checked
- `ok-to-test` matches `pkg/labels.OkToTest` exactly, which `pkg/plugins/trigger` checks to treat a PR as trusted for CI — label achieves the stated intent.
- Both `area/dependency` and `ok-to-test` already exist as labels on `kubernetes-sigs/prow` (confirmed via `gh api repos/kubernetes-sigs/prow/labels`) — no risk of Dependabot referencing nonexistent labels.
- YAML placement/indentation of `labels:` is correct per Dependabot's schema (sibling of `schedule`/`groups`).
- Repo's `dependabot.yaml` has only one ecosystem entry (`gomod`); no separate npm entry to also update — pre-existing state, not a regression from this diff.

## Open questions
(none)
