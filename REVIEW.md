---
pr: kubernetes-sigs/prow#886
title: "chore(deps): bump github.com/maxbrunsfeld/counterfeiter/v6 from 6.4.1 to 6.12.2"
head_sha: 93f1f7d8574e40ffe4b29422225220fddf9a5553
base: main
reviewed_at: 2026-09-01T16:56:15Z
verdict: approve
---

## Verdict

Approve. This is a dependency-only update of a tagged, well-soaked fake-code generator used only during development; Prow's Go 1.26.4 baseline exceeds the generator's Go 1.25 requirement.

## What this PR does

- Updates `github.com/maxbrunsfeld/counterfeiter/v6` from `v6.4.1` to `v6.12.2`.
- Updates the corresponding module checksums and the generator's transitive checksum entries.
- Does not modify Prow source, generated fakes, tests, or runtime dependencies.

## Findings

None.

## Checked

- Classified the two-file `go.mod`/`go.sum` diff as dep-only.
- Confirmed `v6.12.2` is a tagged release from 2026-03-12, 173 days old at review; it is not a pseudo-version.
- Confirmed direct imports occur only in `pkg/plugins/cherrypickapproved/tools/tools.go:26` and `pkg/plugins/testfreeze/tools/tools.go:26`, with three `go:generate` call sites: `pkg/plugins/cherrypickapproved/cherrypickapproved.go:77`, `pkg/plugins/testfreeze/testfreeze.go:103`, and `pkg/plugins/testfreeze/checker/checker.go:83`.
- Reviewed substantive upstream changes: generic functions/interfaces and custom generic constraints, Go 1.23 aliases, and a generated `Invocations()` deadlock fix. Prow's three generation targets are non-generic; no runtime or sensitive code path imports the tool.
- Confirmed the new generator requires Go 1.25 and the repository declares Go 1.26.4.
- `go mod verify` passed. The PR's required `pull-prow-verify-lint` check passed.

## Open questions

None.

## Dependency followups

Examined on 2026-10-21. PR #886 merged as `bdd51da2e19d3a691d505223e76301442f656d84`; execute accepted work on the current default branch, checking whether each opportunity has already been addressed there. Accepted: 2. Skipped: 0.

### Scope examined

- Direct dependency: `github.com/maxbrunsfeld/counterfeiter/v6 v6.4.1 → v6.12.2`; reused the prior release-range analysis and examined the upstream generator deadlock fix and Go-tool migration documentation.
- Transitive checksum changes: `github.com/onsi/gomega v1.39.0 → v1.39.1` (dependency updates and Go-baseline change; no project imports), and `github.com/onsi/ginkgo v1.12.1 → v1.11.0` (older transitive checksum selection; no project imports). Neither offers a project call-site migration.
- Other removed historical `/go.mod` checksums are pruning, not evidence of selected-version upgrades. Existing selected versions of `golang.org/x/net`, `golang.org/x/sys`, `golang.org/x/text`, and `golang.org/x/tools` did not change in this PR.
- Generic-interface, generic-constraint, and alias support does not suggest a concrete migration for Prow's three current generator targets.

### 1. Regenerate checked-in fakes to receive the Invocations deadlock fix

- status: accepted
- category: api-upgrade
- necessity: could — preventive maintenance; no current `.Invocations()` callers were found in the non-vendor Go source.
- unlocked_by: `github.com/maxbrunsfeld/counterfeiter/v6 v6.4.1 → v6.12.2`
- where: `pkg/plugins/cherrypickapproved/cherrypickapprovedfakes/fake_impl.go:342`, `pkg/plugins/testfreeze/testfreezefakes/fake_verifier.go:174`, `pkg/plugins/testfreeze/checker/checkerfakes/fake_checker.go:485`.
- upstream: [Counterfeiter commit 386d8831](https://github.com/maxbrunsfeld/counterfeiter/commit/386d8831d95d053807d08b1620e0347c186f7673), "avoid locking method-specific mutexes in Invocations() to avoid deadlock". All three checked-in fakes retain the old locking pattern; a module bump alone does not update generated source.

```text
In kubernetes-sigs/prow, now that PR #886 — "chore(deps): bump github.com/maxbrunsfeld/counterfeiter/v6 from 6.4.1 to 6.12.2" — merged as bdd51da2e19d3a691d505223e76301442f656d84, regenerate the three Counterfeiter fakes to adopt its Invocations() deadlock fix.

Start from the current default branch in an isolated worktree. Check first whether this work is already complete; do not revert newer generator changes or downgrade dependencies. The reviewed revision has old method-specific locks in pkg/plugins/cherrypickapproved/cherrypickapprovedfakes/fake_impl.go:342, pkg/plugins/testfreeze/testfreezefakes/fake_verifier.go:174, and pkg/plugins/testfreeze/checker/checkerfakes/fake_checker.go:485. Upstream commit 386d8831d95d053807d08b1620e0347c186f7673 removes these locks from the generator template to prevent lock-order deadlocks. There are no known Invocations() callers in the reviewed project source, so describe this as preventive maintenance, not a reproduced Prow failure.

Run the existing generation directives only for ./pkg/plugins/cherrypickapproved, ./pkg/plugins/testfreeze, and ./pkg/plugins/testfreeze/checker using the repository-pinned generator. Inspect the generated diff; do not hand-edit generated locking code.

Acceptance criteria: each regenerated Invocations() retains its invocation-map read lock without acquiring method-specific mutexes; a second generation produces no further changes; go test ./pkg/plugins/cherrypickapproved ./pkg/plugins/testfreeze/... passes. Report any additional generator-produced differences and why they are needed.

Scope: only these three generated fake files and necessary generated output. Do not alter plugin behavior, interfaces, dependency versions, or tool registration. A separate accepted followup covers native Go tool registration.
```

### 2. Adopt native Go tool tracking for Counterfeiter

- status: accepted
- category: simplification
- necessity: could — existing tool tracking works; this replaces legacy bookkeeping with the documented native mechanism.
- unlocked_by: `github.com/maxbrunsfeld/counterfeiter/v6 v6.4.1 → v6.12.2`, whose updated documentation adopts Go 1.24+ tooling; the mechanism itself is a Go feature, not a new Counterfeiter API.
- where: `go.mod:50`; `pkg/plugins/cherrypickapproved/tools/tools.go:26`; `pkg/plugins/testfreeze/tools/tools.go:26`; generation directives at `pkg/plugins/cherrypickapproved/cherrypickapproved.go:77`, `pkg/plugins/testfreeze/testfreeze.go:103`, and `pkg/plugins/testfreeze/checker/checker.go:83`.
- upstream: [Counterfeiter's v6.12.2 README](https://github.com/maxbrunsfeld/counterfeiter/blob/v6.12.2/README.md#using-counterfeiter) recommends `go get -tool github.com/maxbrunsfeld/counterfeiter/v6` and `go tool counterfeiter -generate`; [commit 253faf58](https://github.com/maxbrunsfeld/counterfeiter/commit/253faf58a7bdb0adbe81fc6607d2820369f07067) documents the migration. Prow already declares Go 1.26.4.

```text
In kubernetes-sigs/prow, now that PR #886 — "chore(deps): bump github.com/maxbrunsfeld/counterfeiter/v6 from 6.4.1 to 6.12.2" — merged as bdd51da2e19d3a691d505223e76301442f656d84, simplify Counterfeiter tracking using native Go tool dependencies.

Start from the current default branch in an isolated worktree and check whether this migration is already complete. Preserve the current pinned Counterfeiter version; do not downgrade it to the historical PR version. The v6.12.2 upstream README recommends go get -tool github.com/maxbrunsfeld/counterfeiter/v6 and go tool counterfeiter -generate (documentation commit 253faf58a7bdb0adbe81fc6607d2820369f07067). The reviewed Prow revision declares Go 1.26.4, which supports this Go 1.24 feature.

Register github.com/maxbrunsfeld/counterfeiter/v6 with a tool directive in the root go.mod, preserving its existing requirement/version. Replace the go:generate go run github.com/maxbrunsfeld/counterfeiter/v6 -generate directives in pkg/plugins/cherrypickapproved/cherrypickapproved.go:77, pkg/plugins/testfreeze/testfreeze.go:103, and pkg/plugins/testfreeze/checker/checker.go:83 with go:generate go tool counterfeiter -generate. Remove the dedicated Counterfeiter-only tracking files pkg/plugins/cherrypickapproved/tools/tools.go and pkg/plugins/testfreeze/tools/tools.go after confirming no other tooling relies on them.

Acceptance criteria: tool registration resolves the existing pinned Counterfeiter version; all three generation directives run successfully; repeat generation is stable; go test ./pkg/plugins/cherrypickapproved ./pkg/plugins/testfreeze/... passes. Review any module checksum changes and keep them limited to this migration. Coordinate with the separate fake-regeneration followup so generated changes are accounted for once.

Scope: Counterfeiter registration, its two dedicated tracking files, and the three generation directives only, plus unavoidable checksum/generated changes needed for validation. Do not migrate unrelated tools, bump dependency versions, or change plugin interfaces or runtime behavior.
```
