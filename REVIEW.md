---
pr: kubernetes-sigs/prow#738
title: "invalidcommitmsg: add config surface for fixup commit checking"
head_sha: cdf377c5a45273b181f7f6e8416f72fa7f43696c
base: main
reviewed_at: 2026-06-02T09:02:10Z
verdict: request-changes
---

## Findings

### [blocking] issueClosingKeywords registered in config but never consulted at runtime
- where: `pkg/plugins/config.go:1529`, `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:93`
- concern: `validInvalidCommitMsgChecks` includes `"issueClosingKeywords"`, it passes validation, and the help text documents it as disableable — but `IsCheckDisabled("issueClosingKeywords")` is never called in `handle()`. The `CloseIssueRegex` checks at lines 115 and 124 run unconditionally. An operator who configures `{name: issueClosingKeywords, disabled: true}` gets no effect, silently. Either gate the checks in `handle()` behind `!cfg.IsCheckDisabled("issueClosingKeywords")`, or remove `issueClosingKeywords` from the valid set until it is implemented. Flagged by all three reviewers.
- excerpt: |
    var validInvalidCommitMsgChecks = sets.New[string]("fixupPrefix", "issueClosingKeywords")
    // handle() only ever calls:
    checkFixup := !cfg.IsCheckDisabled("fixupPrefix")
    // IsCheckDisabled("issueClosingKeywords") is never called

### [blocking] fixup check default silently flipped from opt-in to opt-out
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:92-93`
- concern: Old code `os.Getenv("ENABLE_FIXUP_CHECK") == "true"` defaulted to false — opt-in. New code: `InvalidCommitMsgFor` returns `&InvalidCommitMsg{}` when no config exists; `IsCheckDisabled("fixupPrefix")` on an empty struct returns `false` (not found → not disabled); `checkFixup = !false = true`. Fixup checking is now ON by default for any repo without an explicit `invalid_commit_msg` block. Any deployment that relied on not setting `ENABLE_FIXUP_CHECK` will silently start labeling PRs with fixup commits. Must either restore opt-in semantics (return `true` from `IsCheckDisabled` when check is absent) or explicitly commit to opt-out and document it with a migration note. Flagged by all three reviewers.
- excerpt: |
    // Old:
    checkFixup := os.Getenv("ENABLE_FIXUP_CHECK") == "true"
    // New (no config → &InvalidCommitMsg{} → IsCheckDisabled returns false):
    cfg := config.InvalidCommitMsgFor(org, repo)
    checkFixup := !cfg.IsCheckDisabled("fixupPrefix")  // = !false = true

### [nit] stale comment copied from TriggerFor
- where: `pkg/plugins/config.go:1133`
- concern: "triggers" is wrong noun — copied verbatim from `TriggerFor`.
- excerpt: |
    // Prioritize repo level triggers over org level triggers.

### [nit] test case names still reference removed feature flag
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg_test.go:194,202`
- concern: Cases named "feature flag enabled/disabled" — the env-var feature flag is gone. Should be "fixup check config enabled/disabled" or similar.

### [nit] validateRepoDupes error message hardcodes "welcome" — now user-facing
- where: `pkg/plugins/config.go` (pre-existing)
- concern: `validateRepoDupes(c.InvalidCommitMsg)` will produce `The repo "x/y" is duplicated in the 'welcome' plugin configuration.` for `invalid_commit_msg` duplicates. Pre-existing bug but now concretely user-facing via this PR's new call. Worth a follow-up issue.

### [question] is the opt-out default for fixup checking intentional policy?
- where: `pkg/plugins/invalidcommitmsg/invalidcommitmsg.go:92-93`
- concern: If the flip to default-enabled is a deliberate policy decision (making Prow enforce fixup commit hygiene by default), it needs to be called out explicitly in the PR description, plugin help text, and release notes. If it's accidental, it's a bug. The PR description doesn't address it either way.

## Checked
- Two-pass org/repo precedence lookup exactly matches `LgtmFor`/`ApproveFor` pattern; test covers the case where org entry precedes repo entry in slice
- Loop variable `&cfg` returned from range loop is safe — variable escapes to heap on address-take; same pattern as `LgtmFor`/`DcoFor`
- `validateRepoDupes` integration correct; `getRepos()` satisfies `ListableRepos` interface
- `validateInvalidCommitMsg` accumulates all errors via `utilerrors.NewAggregate` (not fail-fast) — consistent with file
- Removal of `"os"` import from `invalidcommitmsg.go` correct — no remaining usages
- Test refactoring from `t.Setenv("ENABLE_FIXUP_CHECK", ...)` to config struct is complete and clean
- `omitempty` on `InvalidCommitMsg` field means no existing `plugins.yaml` will fail to parse on upgrade
- Validation is wired into `Configuration.Validate()` at load time — config errors surface at startup, not runtime

## Open questions
- Is the fixup checking default-on behavior intentional? If yes, document it and provide a migration note. If no, restore opt-in semantics.
- Is `issueClosingKeywords` meant to be wired up in this PR or a future one? If future, remove it from the valid set now and add a TODO comment at the `handle()` call site.
- Should `ENABLE_FIXUP_CHECK` removal include a startup log warning for deployments that still have it set in their environment manifests?
