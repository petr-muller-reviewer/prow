---
pr: kubernetes-sigs/prow#1018
title: "chore(deps): bump github.com/maxbrunsfeld/counterfeiter/v6 from 6.12.2 to 6.14.0"
head_sha: a229510b78b0ee231651bf874a4fb14c957c6aff
base: main
reviewed_at: 2026-10-22T00:30:25Z
verdict: approve
---

# Review

## Verdict

**Approve.** The bump is safe to take now: the new release is 26 days old, has no OSV advisories, and Prow uses `counterfeiter` only as a build-tagged test-double generator rather than as a runtime dependency. The one follow-up is to regenerate the three affected fakes so the committed generated code matches the new generator.

## What this PR does

- Classification: **dep-only**. The PR changes only `go.mod` and `go.sum`; it does not touch project source.
- Bumps `github.com/maxbrunsfeld/counterfeiter/v6` from `v6.12.2` to `v6.14.0` in `go.mod:49`.
- Updates transitive lockfile entries for `golang.org/x/tools` (`v0.49.0` → `v0.50.0`) and `github.com/onsi/gomega` (`v1.39.1` → `v1.43.1`).
- Leaves the three generated fakes unchanged even though `v6.14.0` changes generated fake output.

## Findings

### [should-fix] Regenerate the counterfeiter fakes with the new generator

- where: `pkg/plugins/testfreeze/testfreezefakes/fake_verifier.go:12`, `pkg/plugins/testfreeze/checker/checkerfakes/fake_checker.go:13`, `pkg/plugins/cherrypickapproved/cherrypickapprovedfakes/fake_impl.go:11`
- `v6.14.0` changes the generated fake API and output: per-method argument records become exported struct types, fakes gain `Args()` and `CallOrder()`, and the race in void methods is fixed. Regenerating the three fakes produces **262 insertions and 166 deletions** across those files.
- The current fakes still compile and the affected package tests pass, so this is repository hygiene rather than a runtime break. Still, the PR bumps the generator without updating the code that generator owns.
- Regenerating in a scratch worktree and running `go test ./pkg/plugins/testfreeze/... ./pkg/plugins/cherrypickapproved/...` passes.

## Checked

- **Freshness:** `v6.14.0` was tagged on **2026-09-26**, 26 days before this review. It is a real tagged release, not a pseudo-version.
- **Usage:** Direct tool dependency only. Prow imports it solely from the build-tagged `tools` packages at `pkg/plugins/testfreeze/tools/tools.go:26` and `pkg/plugins/cherrypickapproved/tools/tools.go:26`, and uses `go:generate` directives in `pkg/plugins/testfreeze/testfreeze.go:103`, `pkg/plugins/testfreeze/checker/checker.go:83`, and `pkg/plugins/cherrypickapproved/cherrypickapproved.go:77`. There are no runtime imports.
- **Exposure:** Light and test-only. The dependency generates three fakes in two plugin areas; it is not threaded through production code.
- **Vulnerabilities:** OSV reports no advisories for `v6.12.2` or `v6.14.0`. `govulncheck` reports no counterfeiter findings at either the merge-base or PR head. The pre-existing findings in `x/net`, `x/crypto`, and `tektoncd/pipeline` are unchanged by this PR.
- **Changelog:** `v6.13.0` adds same-package and test-package fake generation, global `-generate` flags, and large performance improvements. `v6.14.0` adds `Args()` and `CallOrder()`, fixes a race in void methods, and updates transitive dependencies.
- **Compatibility:** `v6.13.0` requires Go 1.26; the repository already uses Go 1.26.4 in `go.mod:3` and pins Go 1.26.0 in `.go-version`.
- **Indirect churn:** `golang.org/x/tools` and `github.com/onsi/gomega` are transitive and not directly imported by Prow source. `x/tools` is reached through `golang.org/x/lint`, and `gomega` through controller-runtime test code.
- **Validation:** `go mod tidy -diff` is clean. The PR’s unit, integration, lint, and image-build checks pass.

## Open questions

- None.

## Dependency followups

1. **Accepted: 1 / Skipped: 0.**

### Regenerate counterfeiter fakes after the v6.14.0 bump

- **Unlocked by:** `github.com/maxbrunsfeld/counterfeiter/v6` `v6.12.2` → `v6.14.0`
- **Category:** simplification
- **Where:** `pkg/plugins/testfreeze/testfreezefakes/fake_verifier.go:12`, `pkg/plugins/testfreeze/checker/checkerfakes/fake_checker.go:13`, `pkg/plugins/cherrypickapproved/cherrypickapprovedfakes/fake_impl.go:11`
- **Necessity:** should — keeps committed generated code aligned with the generator version and picks up the void-method race fix.
- **Prompt:**

```text
In kubernetes-sigs/prow, now that PR #1018 bumped github.com/maxbrunsfeld/counterfeiter/v6 from v6.12.2 to v6.14.0, regenerate the three affected fakes so the committed generated code matches the new generator.

Files:
- pkg/plugins/testfreeze/testfreezefakes/fake_verifier.go
- pkg/plugins/testfreeze/checker/checkerfakes/fake_checker.go
- pkg/plugins/cherrypickapproved/cherrypickapprovedfakes/fake_impl.go

Run:
go generate ./pkg/plugins/testfreeze/... ./pkg/plugins/cherrypickapproved/...

Acceptance criteria:
- The diff is limited to the three generated fake files listed above.
- The new generated code compiles and the affected package tests pass:
  go test ./pkg/plugins/testfreeze/... ./pkg/plugins/cherrypickapproved/...

Scope guard:
- Do not adopt the new Args() or CallOrder() APIs in this followup.
- Do not change production code or unrelated tests.
```

No other opportunities were identified. The new `Args()` and `CallOrder()` fake APIs were examined, but the four existing `ArgsForCall` call sites are already minimal and there are no call-order assertions to simplify.
