---
pr: kubernetes-sigs/prow#778
title: "Add /override-sticky command for persistent overrides"
head_sha: a28708fb8055cd4b5376a84abfcec2443faecdd8
base: main
reviewed_at: 2026-07-03T11:30:53Z
verdict: approve
refresh_log:
  - from: e13475980c21dd371d222b70e6a21011652bfc63
    to: a28708fb8055cd4b5376a84abfcec2443faecdd8
    summary: "Author addressed 6 of 8 findings: extracted isAuthorized helper, sequential multi-command dispatch, broadened /override-cancel to all overrides, description length test, fixed error messages"
---

## Summary

Adds `/override-sticky` and `/override-cancel` commands. Sticky overrides embed `[prow:skip-retest]` in the GitHub status description so Tide treats them as permanently passing for the current HEAD SHA. Regular `/override` now embeds baseSHA via `ContextDescriptionWithBaseSha`, fixing the Sinker-reaping expiry bug. Tide's `prowJobsFromContexts` gains an `IsSkipRetest` check. No config schema changes, no new dependencies, low deployment risk.

Since previous review (a28708fb):
- Author extracted `isAuthorized()` helper, eliminating the auth duplication
- Dispatch now calls all three handlers sequentially instead of early-return if/else-if
- `/override-cancel` broadened to work on all overrides (checks `"Overridden by"` instead of sentinel)
- Error messages now reflect actual command name via `cmdName` variable
- Added `TestStickyDescriptionFitsGitHubLimit` test for description length

## Review posted

Review comments posted on 2026-07-01T14:16:00Z by petr-muller. Author responded on 2026-07-01T17:24:37Z and pushed a28708fb addressing most feedback.

## Findings

### [should-fix] Regular /override should reuse baseSHA from the overridden context
- where: `pkg/plugins/override/override.go:513`
- concern: `baseSHAGetter()` fetches the current base branch HEAD at override time. For regular `/override`, this is wrong — it should reuse the baseSHA already embedded in the overridden context's description (via `BaseSHAFromContextDescription`). The override just changes the verdict, it shouldn't claim the job ran against a different base. If the original context has no baseSHA, fall back to the current base as today.
- posted: yes (petr-muller, override.go:513)
- status: OPEN — not addressed in a28708fb

### [nit] Operator precedence ambiguity in Tide condition
- where: `pkg/tide/tide.go:1056`
- concern: Correct per Go rules but reads ambiguously. Explicit parentheses recommended.
- status: OPEN — not addressed in a28708fb
- excerpt: |
    if config.IsSkipRetest(desc) || baseSHAForContext != "" && baseSHAForContext == baseSHA {

## Resolved

### [should-fix] Mixed command types in one comment silently dropped
- resolved in: a28708fb
- how: Dispatch now calls all three handlers sequentially, each returns early if no matching regex. All command types in one comment are processed.

### [should-fix] Authorization logic duplicated between handle() and handleOverrideCancel()
- resolved in: a28708fb
- how: Extracted `isAuthorized()` helper used by both `handle()` and `handleOverrideCancel()`.

### [should-fix] /override-cancel should also check for "Overridden by" in description
- resolved in: a28708fb
- how: Cancel now checks `strings.Contains(status.Description, "Overridden by")` instead of `IsSkipRetest`.

### [should-fix] /override-cancel semantics unclear — should it also work on non-sticky overrides?
- resolved in: a28708fb
- how: Cancel now works on all overrides (any status with "Overridden by" in description). Help text updated to say "Removes overrides by setting the status back to failure. Works on both regular and sticky overrides."

### [should-fix] Add description length guard
- resolved in: a28708fb
- how: Added `TestStickyDescriptionFitsGitHubLimit` that builds a sticky description with max-length username (39 chars) and 40-char SHA, asserts it fits in 140 chars and preserves the sentinel.

### [nit] Hardcoded error message doesn't reflect actual command
- resolved in: a28708fb
- how: Error messages now use `cmdName` variable (`"/override"` or `"/override-sticky"`).

## Followup ideas (posted)

- Consider whether synthetic presubmit ProwJob creation (override.go:540) is still needed now that statuses embed baseSHA — could be removed as a followup.
- Add a documentation page for the override plugin at https://docs.prow.k8s.io/docs/components/plugins/ covering command behavior, interaction with Tide, and the difference between standard and sticky overrides.

## Checked
- Operator precedence in `tide.go:1056` — `&&` binds tighter than `||`, correct parse
- `ContextDescriptionWithBaseSha` truncation — sentinel is 18 chars, last 43 chars always preserved, sentinel survives
- Other Tide code paths (`unsuccessfulContexts`, `filterPR`, `isRetestEligible`) operate on status state (SUCCESS), no sentinel awareness needed
- `fakeClient.CreateStatus` test changes — removed guards needed for `/override-cancel`, old special case was redundant
- Cross-file callers of `handle()` — only called from `handleGenericComment`, signature change fully propagated
- No config schema changes, no new required fields, no CLI flags or env vars changed
- Rollback is safe — graceful degradation
- No ordering dependency between override plugin and Tide upgrades
- `overrideRe` does NOT match `/override-sticky` — the optional capture group requires a literal space after `/override`
- New `isAuthorized` helper correctly covers all three auth paths (admin, team, topLevelOwner)
- Sequential dispatch in `handleGenericComment` is correct — each handler returns nil if no matching regex, errors propagate

## Open questions
- Are synthetic presubmit ProwJobs still needed now that statuses embed baseSHA?
