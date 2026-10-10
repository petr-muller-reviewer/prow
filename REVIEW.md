---
pr: kubernetes-sigs/prow#771
title: "branchprotector: add require_signed_commits config"
head_sha: 892e2930e347a01055d180247b348aa2d46cbc82
base: main
reviewed_at: 2026-07-26T23:34:21Z
verdict: approve
gate:
  decision: merged
  gated_at: 2026-06-24T12:25:33Z
  gated_head_sha: 892e2930e347a01055d180247b348aa2d46cbc82
  reviewed_head_sha: 892e2930e347a01055d180247b348aa2d46cbc82
refresh_log:
  - from_head_sha: 892e2930e347a01055d180247b348aa2d46cbc82
    to_head_sha: 892e2930e347a01055d180247b348aa2d46cbc82
    at: 2026-07-26T23:34:21Z
    summary: No code changes. petr-muller approved ("lgtm, thanks!") on 2026-06-24T12:27:42Z; PR labeled lgtm+approved and merged by kubernetes-prow bot at 2026-06-24T12:28:24Z (merge commit 2b5fea27a177c767160452ba75dba978a88d8d63) without the should-fix findings being addressed.
---

## Gate

**Decision: merged**

PR merged 2026-06-24T12:28:24Z (merge commit `2b5fea27a177c767160452ba75dba978a88d8d63`) after petr-muller approved with "lgtm, thanks!" and the bot applied `lgtm`+`approved` labels. No code changes occurred between the prior hold and merge — the two should-fix findings below were **not** addressed before merge; the reviewer accepted them as-is.

### Unaddressed findings (merged as-is)
- **[should-fix] applySeparateRequests fires after UpdateBranchProtection failure** (`protect.go:204-210`): unchanged at merge. Recommend adding `continue` after the error as a followup.
- **[should-fix] TestConfigureBranches lacks coverage for applySeparateRequests path**: no new test cases added at merge; tracked as a followup below.

### Merge risk (Area 2)
- **`RepositoryClient` interface widened** (`pkg/github/client.go:174-175`): two new exported methods added. Technically backward-incompatible for external implementors. In practice, `RepositoryClient` is not referenced by name outside `pkg/github/client.go` in production code — it's embedded into the aggregate `Client` interface. Risk is low; this follows the standard pattern for provider-side interfaces in this codebase.
- **Config change**: additive `*bool` with `omitempty` — no impact on existing configs.
- **Behavioral change**: opt-in only — no effect on existing deployments that don't set `require_signed_commits`.

### What unblocks
Merged — no longer applicable. The two should-fix findings were accepted as-is at merge; see the Followups section for post-merge cleanup prompts.

## Findings

### [should-fix] applySeparateRequests fires after UpdateBranchProtection failure
- where: `cmd/branchprotector/protect.go:204-210`
- concern: When `UpdateBranchProtection` fails, execution falls through to `applySeparateRequests`. The signature endpoint requires branch protection to be enabled, so the POST/DELETE will also fail (or leave partial state). Adding `continue` after the error would skip redundant failing calls. Three independent reviewers converged on this.
- excerpt: |
    if err := p.client.UpdateBranchProtection(u.Org, u.Repo, u.Branch, *u.Request); err != nil {
        p.errors.add(fmt.Errorf("update %s/%s=%s protection to %v failed: %w", u.Org, u.Repo, u.Branch, *u.Request, err))
    }
    
    if u.Separate != nil {
        p.applySeparateRequests(u.Org, u.Repo, u.Branch, u.Separate)
    }

### [should-fix] TestConfigureBranches lacks coverage for applySeparateRequests path
- where: `cmd/branchprotector/protect_test.go` (TestConfigureBranches)
- concern: No test case sends a `requirements` with `Separate` through the channel. The actual enable/disable calls and error accumulation from `applySeparateRequests` are untested. Adding a case with `Separate: &separateRequests{RequireSignedCommits: &yes}` and one with `branch: "error"` would close the gap.

### [nit] TestProtect error diagnostic omits Separate field
- where: `cmd/branchprotector/protect_test.go:~1660`
- concern: On mismatch, `ObjectDiff` compares `a.Request` vs `e.Request` but not `a.Separate` vs `e.Separate`. If `Separate` fields differ, the test fails with unhelpful output.

### [question] Silent no-op when require_signed_commits is set without protect
- where: `cmd/branchprotector/protect.go:494`
- concern: Setting `require_signed_commits: true` without `protect: true` anywhere in the hierarchy causes `UpdateBranch` to bail at line 494 (`bp.Protect == nil`). The setting is silently ignored. Consistent with other sub-settings (`allow_force_pushes`, etc.) but could confuse users. Worth a log warning or documentation note?

## Checked
- `equalSeparateRequests` nil safety: nil sep, nil state, nil RequireSignedCommits all handled correctly
- `Policy.Apply()` merges `RequireSignedCommits` via `selectBool`; `defined()` includes it
- `protect: false` path: `RemoveBranchProtection` + `continue` correctly skips `applySeparateRequests`
- GitHub API v2022-11-28 includes `required_signatures` in GET branch protection response (preview header graduated); equality check reads correct current state
- HTTP status codes: 200 for POST enable, 204 for DELETE disable, match GitHub API docs
- `FakeClient` in `fakegithub.go` has no compile-time `RepositoryClient` assertion; missing methods don't break compilation
- Config compatibility: new `*bool` with `omitempty` means existing configs parse unchanged, nil = no API calls
- No new permissions needed: `required_signatures` endpoint requires same admin access as `UpdateBranchProtection`
- Rollback safe: removing field stops managing the setting but won't actively disable it on GitHub

## Open questions
- Should `applySeparateRequests` be skipped when `UpdateBranchProtection` fails? Three reviewers flagged this independently.
- Would a warning log when `require_signed_commits` is set without `protect: true` be helpful, or is silent consistency with other sub-settings the right call?

## Followups

Selection: 4 accepted, 1 skipped (signed-commit configuration documentation).
These are post-merge followups to PR #771, merged as `2b5fea27a177c767160452ba75dba978a88d8d63`. Work on the current default branch and check whether subsequent changes already address each task.

### error-handling: Skip signature changes after a main protection update fails
- category: error-handling
- necessity: should
- where: `cmd/branchprotector/protect.go:204-210`, `cmd/branchprotector/protect_test.go`
- why followup: The should-fix finding merged as-is. The signature operation currently proceeds after the main update fails, allowing a partial policy change and potentially redundant errors.

```text
In kubernetes-sigs/prow, following PR #771 — "branchprotector: add require_signed_commits config" (merge commit 2b5fea27a177c767160452ba75dba978a88d8d63), skip signature changes after a failed main protection update.

Work on the current default branch. First check whether subsequent changes already address this task.

In cmd/branchprotector/protect.go, configureBranches currently records an UpdateBranchProtection error and then calls applySeparateRequests. Stop processing that branch's signature operations when the main update fails, retaining the error and continuing with later queued branches. A successful main update must still permit signature changes.

Add a focused regression test in protect_test.go. Make the main update fail independently of the signature endpoints, record signature calls, and assert that neither enable nor disable is called after that failure. Cover both desired signature values, verify the retained main-update error, and verify that the next queued branch is still processed. Do not rely solely on the fake's existing branch == "error" behavior, which fails multiple operations together.

Acceptance criteria: go test ./cmd/branchprotector/... passes. A failed main update records its error without invoking either signature endpoint; later branches continue normally.

Scope: Limit changes to cmd/branchprotector/protect.go and protect_test.go. Preserve removal behavior and successful updates. Do not change policy configuration or GitHub client methods.
```

### tests: Exercise signature operations through configureBranches
- category: tests
- necessity: should
- where: `cmd/branchprotector/protect_test.go:205-303`
- why followup: The should-fix finding merged as-is. Existing tests inspect emitted requirements and equality, but do not exercise signature calls or their errors through configureBranches.

```text
In kubernetes-sigs/prow, following PR #771 — "branchprotector: add require_signed_commits config" (merge commit 2b5fea27a177c767160452ba75dba978a88d8d63), add execution-path coverage for signature operations in TestConfigureBranches, in cmd/branchprotector/protect_test.go.

Work on the current default branch. First check whether subsequent changes already provide this coverage.

Add cases with successful main protection updates followed by RequireSignedCommits true and false, asserting that the expected enable and disable calls occur. Add cases with Separate nil and RequireSignedCommits nil, asserting that neither signature endpoint is called. Cover protection removal with a Separate value present and verify that removal skips signature operations.

Extend the fake client so enable and disable can fail independently while the main update succeeds. Assert that each signature error is retained and subsequent queued branches are still processed. Record and compare the signature operations as well as existing main update/removal results.

A separate accepted followup skips signature operations after a main-update failure. Keep these tests compatible with that behavior: isolate signature failures instead of requiring the previous two-error cascade from branch == "error". Give requests for normal signature-operation cases a non-nil main Request under the existing queue semantics, where Request == nil means removal.

Acceptance criteria: go test ./cmd/branchprotector/... passes. Tests exercise enable, disable, unspecified settings, removal, and independently injected signature errors through configureBranches.

Scope: Change only cmd/branchprotector/protect_test.go. Preserve production behavior. Do not duplicate the existing equalSeparateRequests tests or the focused main-update failure regression test from the error-handling followup.
```

### test-diagnostics: Include Separate in TestProtect mismatch output
- category: test-diagnostics
- necessity: could
- where: `cmd/branchprotector/protect_test.go:1660`
- why followup: The nit merged as-is. TestProtect compares complete requirements values but reports differences only in Request, obscuring mismatches in Separate.

```text
In kubernetes-sigs/prow, following PR #771 — "branchprotector: add require_signed_commits config" (merge commit 2b5fea27a177c767160452ba75dba978a88d8d63), improve TestProtect's mismatch diagnostic in cmd/branchprotector/protect_test.go.

Work on the current default branch. First check whether the diagnostic has already been corrected.

After the existing fixup calls, the test compares complete requirements values, but its failure message diffs only a.Request and e.Request. Update that diagnostic to diff the complete normalized requirements values, including Separate and RequireSignedCommits, using an existing repository diff helper. Preserve matching, normalization, and pass/fail behavior.

Acceptance criteria: go test ./cmd/branchprotector/... passes. The failure diagnostic includes Separate when actual and expected signature settings differ.

Scope: Change only cmd/branchprotector/protect_test.go. Do not alter production code, test matching, or expected behavior. Do not add a separate diagnostic test harness.
```

### efficiency: Avoid redundant API calls while preserving protection removal
- category: efficiency
- necessity: could
- where: `cmd/branchprotector/protect.go:196-210,535-555`, `cmd/branchprotector/protect_test.go`
- why followup: The new independent signature endpoint makes it useful to reconcile each component separately. The previous handoff treated a nil Request as an unchanged main policy, but configureBranches currently interprets it as removal; that ambiguity must be resolved first.

```text
In kubernetes-sigs/prow, following PR #771 — "branchprotector: add require_signed_commits config" (merge commit 2b5fea27a177c767160452ba75dba978a88d8d63), avoid redundant main-protection and signature API calls when only one component changes.

Work on the current default branch. First check whether subsequent changes already implement this optimization.

UpdateBranch currently combines equalBranchProtections and equalSeparateRequests; a difference in either queues both components. Before splitting those checks, make the queued requirements distinguish keeping, updating, and removing main protection. Today configureBranches treats Request == nil as removal: simply setting Request nil for an unchanged main policy would delete protection and skip the signature change. Choose a small explicit representation that avoids that ambiguity.

Queue and execute only the operations required by the detected differences. Preserve protect: false removal, creation of main protection before enabling signatures, and skipping signature operations when a required main update fails. An unchanged main policy with a changed signature setting must perform only the signature operation and must never remove protection. When neither component differs, queue nothing.

Confirm from the GitHub API contract that updating main protection preserves the signature setting before skipping an unchanged signature request; if it does not, retain the necessary signature request and document why.

Add or update TestProtect and TestConfigureBranches cases for main-only changes, signature-only changes, both components changed, neither changed, creation, and removal. Assert intended calls and the absence of unintended calls, including removal. Integrate with the separate accepted error-handling and execution-test followups rather than restoring the prior error cascade.

Acceptance criteria: go test ./cmd/branchprotector/... passes. A signature-only change makes no main PUT or protection DELETE. Main-only changes avoid a signature call where the API contract permits it. Combined changes, creation, removal, and failure ordering retain their intended behavior.

Scope: Keep implementation and tests within cmd/branchprotector/. Do not change GitHub client methods, public config fields, or pkg/github/types.go.
```
