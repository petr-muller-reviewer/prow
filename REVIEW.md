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

## Followups

Post-merge followup for PR #852 (merged as `c8b6c2ce8b130126a73672324a6ced4bd83df8a4`, 2026-08-17). 1 accepted, 2 skipped.

### [automation] Cover npm Dependabot PRs in `.github/dependabot.yaml`
- where: `.github/dependabot.yaml`, `site/package.json`, `site/package-lock.json`
- necessity: should
- mode: deliver
- why: The merged `labels:` block only applies to the `gomod` entry. npm security-update PRs (`#1019` source-map-js, `#839` postcss) still open with Dependabot's defaults (`dependencies`, `javascript`) and no `area/dependency`, and still need a human `/ok-to-test` — so the stated intent holds for Go modules only.
- scope: Add an `npm` update entry for `/site` (tracked lockfile; root `package.json` has none) with `schedule.interval: weekly` and the same two labels; leave the existing `gomod` entry untouched.

Considered and skipped: suppressing trigger's contradictory "waiting for member" comment on bot PRs with pre-applied `ok-to-test`; restoring Dependabot's default `dependencies`/`go` labels.

```text
In `kubernetes-sigs/prow`, following merged PR #852 ("chore: label dependabot PRs with area/dependency and ok-to-test", merge commit c8b6c2ce8b130126a73672324a6ced4bd83df8a4), extend `.github/dependabot.yaml` so npm dependencies get the same Dependabot treatment as Go modules.

What and where:
- `.github/dependabot.yaml` currently has a single `gomod` entry whose `labels:` list grants `area/dependency` and `ok-to-test`.
- `site/package.json` + `site/package-lock.json` are npm-managed but have no Dependabot entry. Recent npm security-update PRs (#1019, #839) still opened with the default `dependencies`/`javascript` labels and no `area/dependency`, so they still need a manual `/ok-to-test`.

Task:
- Add a second update entry to `.github/dependabot.yaml`:

  - package-ecosystem: "npm"
    directory: "/site"
    schedule:
      interval: "weekly"
    labels:
      - "area/dependency"
      - "ok-to-test"

- Use `directory:` (singular) for the npm entry, matching the existing entry's indentation and quoting style.
- Do not add an npm entry for `/`: the root `package.json` has no tracked lockfile, so Dependabot cannot run version updates there.

Acceptance criteria:
- `.github/dependabot.yaml` parses as valid YAML and the new entry is a sibling of the existing `gomod` entry under `updates:`.
- `/site` is covered by an npm entry with both labels; the existing `gomod` entry (directories, groups, labels) is unchanged.
- No other files changed.

Scope guard:
- Do not modify the `gomod` entry's `groups`/`directories`, do not add npm coverage for `/`, and do not change Prow source code or docs.
- Do not push, open a PR, or otherwise publish anything; leave the change local unless explicitly asked.
