---
pr: kubernetes-sigs/prow#898
title: "chore(deps): bump github.com/andygrunwald/go-gerrit from 1.1.1 to 1.2.0"
head_sha: 1e7ec7246542b8705aa984338efd72f6db3e0c61
base: main
reviewed_at: 2026-09-01T00:16:28Z
verdict: approve
---

## Summary

Dependabot dep-only bump: `github.com/andygrunwald/go-gerrit` v1.1.1 → v1.2.0 (direct),
which transitively pulls `github.com/google/go-querystring` v1.1.0 → v1.2.0 (indirect).
Only `go.mod`/`go.sum` change; no project code touched. Already merged 2026-08-26.

- go-gerrit v1.2.0 released 2026-08-22; PR opened/merged only 4 days later — under the
  ~5-day freshness bar, though at +10 days now with no reported fallout.
- go-gerrit is a direct dep with a heavy import surface: `pkg/gerrit/client`,
  `pkg/gerrit/adapter`, `pkg/gerrit/fakegerrit`, `pkg/crier/reporters/gerrit`,
  `pkg/tide/{gerrit,codereview}`, plus integration tests.
- Changelog is purely additive: `WithRunAs` context (X-Gerrit-RunAs header),
  `ChangeInfo.CurrentRevisionNumber` field, new `State`/`Notify`/`NotifyDetails` fields on
  `ReviewerInput`. No CVEs, no behavior changes to existing client logic.
- Release notes flag a breaking-shape gotcha: `ReviewerInput` gained a `State` field
  inserted between `Reviewer` and `Confirmed`, breaking unkeyed struct literals. Grepped
  the project for `ReviewerInput` — zero hits outside vendor, so this doesn't affect us.
- `go-querystring` v1.2.0 is indirect-only, not imported by our code — no further analysis
  needed.

## Findings

None.

## Checked

- `go.mod`/`go.sum` diff — only the two module bumps, no other changes.
- Freshness of go-gerrit v1.2.0 via proxy.golang.org (real tag, not pseudo-version).
- Import surface of both bumped modules (`grep` outside vendor).
- go-gerrit release notes and commit range v1.1.1...v1.2.0 for breaking changes/CVEs.
- Usage of `ReviewerInput` in project code (none) — the one breaking-shape change in the
  release doesn't apply.

## Open questions

None.
