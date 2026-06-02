---
pr: kubernetes-sigs/prow#738
title: "invalidcommitmsg: add config surface for fixup commit checking"
head_sha: 7189547bdfc3d607ec9d4f621f1902e80f9a1347
base: main
reviewed_at: 2026-06-02T23:11:46Z
verdict: request-changes
state: merged
refresh_log:
  - old_sha: cdf377c5a45273b181f7f6e8416f72fa7f43696c
    new_sha: 7189547bdfc3d607ec9d4f621f1902e80f9a1347
    summary: "PR merged; one commit added wiring up issueClosingKeywords check (resolves blocking finding #1); fixup-default polarity finding merged unresolved"
---

## Findings

### [resolved] issueClosingKeywords registered in config but never consulted at runtime
- resolved_in: `7189547bdfc3d607ec9d4f621f1902e80f9a1347`
- resolution: Commit "invalidcommitmsg: make issueClosingKeywords check configurable via plugin config" adds `checkIssueClosing := !cfg.IsCheckDisabled("issueClosingKeywords")` and gates both `CloseIssueRegex` checks (commit body at line 113, PR title at line 122) behind it. Two new test cases added: "issue closing keywords ignored when check disabled" and "issue closing keywords detected when check enabled". Also cleans up test setup to always pass both checks explicitly via `append` rather than conditional slice construction.

### [blocking] fixup check default silently flipped from opt-in to opt-out — merged unresolved
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:92-93`
- concern: Old code `os.Getenv("ENABLE_FIXUP_CHECK") == "true"` defaulted to false — opt-in. New code: `InvalidCommitMsgFor` returns `&InvalidCommitMsg{}` when no config exists; `IsCheckDisabled("fixupPrefix")` on an empty `Checks` slice returns `false` (check not found = not disabled); so `checkFixup = !false = true`. Fixup checking is now ON by default for any repo without an explicit `invalid_commit_msg` block. Any deployment that relied on not setting the env var will silently start labeling PRs with fixup commits. This was flagged by all three reviewers and was not addressed before merge. Operators upgrading should be aware and add explicit config if they want to preserve the old opt-in behavior.
- excerpt: |
    // Old (opt-in, defaults off):
    checkFixup := os.Getenv("ENABLE_FIXUP_CHECK") == "true"
    // New (opt-out, defaults ON when no config):
    cfg := config.InvalidCommitMsgFor(org, repo)
    checkFixup := !cfg.IsCheckDisabled("fixupPrefix")  // = !false = true

### [nit] stale comment copied from TriggerFor
- where: `pkg/plugins/config.go:1133`
- concern: "triggers" is wrong noun — copied verbatim from `TriggerFor`. Not addressed in the merged PR.
- excerpt: |
    // Prioritize repo level triggers over org level triggers.

### [nit] test case names still reference removed feature flag
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg_test.go:194,202`
- concern: Cases named "fixup commit detected when feature flag disabled/enabled" — the env-var feature flag is gone. Not addressed in the merged PR.

### [nit] validateRepoDupes error message hardcodes "welcome" — now user-facing
- where: `pkg/plugins/config.go` (pre-existing)
- concern: `validateRepoDupes(c.InvalidCommitMsg)` produces `The repo "x/y" is duplicated in the 'welcome' plugin configuration.` for `invalid_commit_msg` duplicates. Pre-existing bug now exposed to users of the new config surface. Not addressed in the merged PR.

## Checked
- Two-pass org/repo precedence lookup exactly matches `LgtmFor`/`ApproveFor` pattern; test covers precedence inversion (org entry before repo entry in slice)
- Loop variable `&cfg` returned from range loop is safe — escape analysis allocates on heap; same pattern as `LgtmFor`/`DcoFor`
- `getRepos()` satisfies `ListableRepos` — plugs into generic `validateRepoDupes` without boilerplate
- `validateInvalidCommitMsg` accumulates all errors via `utilerrors.NewAggregate` (not fail-fast) — consistent with file style
- Removal of `"os"` import from `invalidcommitmsg.go` correct
- Test refactoring from `t.Setenv("ENABLE_FIXUP_CHECK", ...)` to config struct is complete and covers all branches
- `omitempty` on `InvalidCommitMsg` field — existing `plugins.yaml` files parse and validate without change on upgrade
- Validation wired into `Configuration.Validate()` at load time — config errors surface at startup, not silently at runtime
- `issueClosingKeywords` now fully wired: both `CloseIssueRegex` call sites gated, new test cases verify disabled/enabled behavior

## Open questions
- The fixup checking default-on behavior was merged without documentation or migration note — should a follow-up issue or release note be filed?
- Should `ENABLE_FIXUP_CHECK` removal include a startup log warning for deployments that still have it set in their environment manifests?
