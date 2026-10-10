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

### tests: Add TestConfigureBranches coverage for applySeparateRequests
- category: tests
- necessity: should
- where: `cmd/branchprotector/protect_test.go:205-303`
- prompt: |
    ```
    In kubernetes-sigs/prow, following PR #771 ("branchprotector: add require_signed_commits config"), add missing test coverage for the `applySeparateRequests` path in `TestConfigureBranches`.

    Context: PR #771 added a `separateRequests` struct and `applySeparateRequests` method to `configureBranches()` in `cmd/branchprotector/protect.go`. The existing `TestConfigureBranches` (protect_test.go:205-303) exercises the `configureBranches()` goroutine by sending `requirements` through a channel and checking `fc.deleted` and `fc.updated` — but no test case includes `Separate` in its requirements, and assertions don't check `fc.signedCommitsEnabled`. The enable/disable calls and error accumulation from `applySeparateRequests` have zero direct test coverage.

    Task:
    1. Add a test case to `TestConfigureBranches` that sends a `requirements` with `Separate: &separateRequests{RequireSignedCommits: &yes}` and a valid branch, then asserts `fc.signedCommitsEnabled` contains the expected key set to `true`.
    2. Add a test case that sends a `requirements` with `Separate: &separateRequests{RequireSignedCommits: &yes}` and `Branch: "error"`, then asserts `errors` count is 2 (one for UpdateBranchProtection, one for EnableCommitSignProtection — the fakeClient returns errors for branch "error").
    3. Add a test case with `RequireSignedCommits: &no` (false) to verify DisableCommitSignProtection is called.
    4. Extend the test assertions block (around line 295-301) to also compare `fc.signedCommitsEnabled` against expected values for cases that set `Separate`.

    Acceptance criteria: `go test ./cmd/branchprotector/...` passes with the new cases. The new cases exercise both the enable and disable paths and the error path of `applySeparateRequests`.

    Out of scope: Do not modify `configureBranches()` itself, do not change `applySeparateRequests`, do not add tests for `equalSeparateRequests` (already covered). Do not touch files outside `cmd/branchprotector/protect_test.go`.
    ```

### efficiency: Avoid redundant API calls when only one component changed
- category: efficiency
- necessity: could
- where: `cmd/branchprotector/protect.go:204-210,543`
- prompt: |
    ```
    In kubernetes-sigs/prow, following PR #771 ("branchprotector: add require_signed_commits config"), eliminate redundant API calls when only one of {main branch protection, separate requests} has changed.

    Context: In `cmd/branchprotector/protect.go`, `UpdateBranch()` (around line 543) uses a combined equality check: `equalBranchProtections(currentBP, req) && equalSeparateRequests(currentBP, sep)`. When either component differs, the entire `requirements` struct (containing both `Request` and `Separate`) is sent through the channel. `configureBranches()` (lines 204-210) then unconditionally calls both `UpdateBranchProtection` and `applySeparateRequests`. This means:
    - If only `require_signed_commits` changed, `UpdateBranchProtection` is called with unchanged settings (no-op PUT).
    - If only main protection changed, `applySeparateRequests` makes a no-op POST/DELETE.
    Both are idempotent but waste GitHub API quota.

    Task:
    1. Split the equality check in `UpdateBranch()` so each component is evaluated independently.
    2. Only populate `Request` in the `requirements` struct when `equalBranchProtections` is false; only populate `Separate` when `equalSeparateRequests` is false. Handle the case where both are nil (skip entirely, as before).
    3. In `configureBranches()`, gate `UpdateBranchProtection` on `u.Request != nil` (it already does this for the remove-protection path, but not for the update path — currently `Request` is always non-nil when `*bp.Protect` is true). Gate `applySeparateRequests` on `u.Separate != nil` (already done).
    4. Update `TestConfigureBranches` to cover the case where only `Separate` is set (no `Request`) — verify only enable/disable is called, not `UpdateBranchProtection`.
    5. Update `TestProtect` to cover the case where main protection matches but `require_signed_commits` differs — verify the emitted `requirements` has `Request` nil and `Separate` non-nil.

    Acceptance criteria: `go test ./cmd/branchprotector/...` passes. When only signed commits state differs, no PUT to the main branch protection endpoint. When only main protection differs, no POST/DELETE to the signatures endpoint.

    Out of scope: Do not change the GitHub client methods, do not modify `pkg/config/`, do not change `pkg/github/types.go`. Keep changes within `cmd/branchprotector/`.
    ```
