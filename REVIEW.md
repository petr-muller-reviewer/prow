---
pr: kubernetes-sigs/prow#934
title: "chore(deps): bump google.golang.org/api from 0.296.0 to 0.297.0"
head_sha: 0560f22619b146cff991705f4b3ea097d59d0ab4
base: main
reviewed_at: 2026-11-08T00:10:04Z
verdict: approve
refresh_log:
  - at: 2026-11-08T00:10:04Z
    old_sha: 0560f22619b146cff991705f4b3ea097d59d0ab4
    new_sha: 0560f22619b146cff991705f4b3ea097d59d0ab4
    summary: "No code change; PR was approved and merged. Recorded merge outcome and post-review activity."
---

## Verdict

approve — This is a narrow, direct Go dependency update. Upstream v0.297.0 only raises `google.golang.org/api`'s minimum Go version from 1.25 to 1.26; Prow already requires Go 1.26.4. The PR has since been approved and merged (2026-09-14T21:55:19Z) with no further code changes; the approval stands.

## What this PR does

- Updates direct dependency `google.golang.org/api` from `v0.296.0` to `v0.297.0` in `go.mod`.
- Replaces the corresponding two module checksums in `go.sum`.
- Takes upstream's Go 1.26 minimum-version change; it does not add code or alter Prow configuration.

Since previous review:
- No commits were added: the PR head is unchanged at `0560f22619b146cff991705f4b3ea097d59d0ab4`, so there is no diff to re-examine.
- The PR was approved (`petr-muller`, 2026-09-14T21:30:41Z; the `approved`/`lgtm` labels followed at 21:30:48Z) and merged by `kubernetes-prow[bot]` at 2026-09-14T21:55:19Z. Dependabot deleted the head branch; `kubevirt-bot` cross-referenced the PR on 2026-09-18.
- No inline review comments and no further review requests; the only other activity is `kubernetes-prow[bot]`'s APPROVALNOTIFIER comment at 21:30:49Z recording the approval. Findings and open questions are unchanged.

## Findings

None.

## Checked

- Classification: dep-only; changed files are `go.mod` and `go.sum` only.
- `v0.297.0` is a tagged release published 2026-09-02 (about 12 days before review), not a pseudo-version.
- The upstream range `v0.296.0...v0.297.0` contains only the Go 1.26 minimum-version change and release commit; no functional, dependency-graph, or security changes were identified.
- The module is direct and imported in 9 Go files (7 production): Cloud Build, Secret Manager, storage/I/O, and test helpers. Those paths involve credentials and cloud APIs, but none of the upstream change affects their behavior.
- Prow's `go.mod` declares `go 1.26.4`, satisfying the new dependency baseline.
- `git diff --check` passes.

## Open questions

None.

## Dependency followups

No opportunities identified.

- Modules examined: `google.golang.org/api` v0.296.0 → v0.297.0 (direct). No transitive module versions changed by this bump (`go.mod`/`go.sum` diff touches only that module).
- Upstream `v0.296.0...v0.297.0` contains exactly two commits (#3724 "Move to go1.26.0 as the lowest supported go version" and the release commit). The release notes list that as the only feature.
- The only changelog entry is a minimum-Go-version change, which Prow already satisfies (`go 1.26.4`); it adds no APIs, deprecates none, and changes no behavior we rely on. Nothing in Prow's imports of `google.golang.org/api` (Cloud Build, Secret Manager, storage/I/O, test helpers) is affected.
