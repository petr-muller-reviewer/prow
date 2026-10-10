---
pr: kubernetes-sigs/prow#778
title: "Add /override-sticky command for persistent overrides"
head_sha: e8e26219d966f3b9dbdaf0ef6f27a439435529b1
base: main
reviewed_at: 2026-07-26T23:33:05Z
verdict: approve
state: merged
refresh_log:
  - from: e13475980c21dd371d222b70e6a21011652bfc63
    to: a28708fb8055cd4b5376a84abfcec2443faecdd8
    summary: "Author addressed 6 of 8 findings: extracted isAuthorized helper, sequential multi-command dispatch, broadened /override-cancel to all overrides, description length test, fixed error messages"
  - from: a28708fb8055cd4b5376a84abfcec2443faecdd8
    to: e8e26219d966f3b9dbdaf0ef6f27a439435529b1
    summary: "Second reviewer (Prucek) raised the same operator-precedence nit and a new 'Overridden by' constant-extraction nit; author addressed both in e8e26219, then PR was lgtm'd, approved, and merged"
---

## Summary

Adds `/override-sticky` and `/override-cancel` commands. Sticky overrides embed `[prow:skip-retest]` in the GitHub status description so Tide treats them as permanently passing for the current HEAD SHA. Regular `/override` now embeds baseSHA via `ContextDescriptionWithBaseSha`, fixing the Sinker-reaping expiry bug. Tide's `prowJobsFromContexts` gains an `IsSkipRetest` check. No config schema changes, no new dependencies, low deployment risk.

Since previous review (a28708fb):
- Author extracted `isAuthorized()` helper, eliminating the auth duplication
- Dispatch now calls all three handlers sequentially instead of early-return if/else-if
- `/override-cancel` broadened to work on all overrides (checks `"Overridden by"` instead of sentinel)
- Error messages now reflect actual command name via `cmdName` variable
- Added `TestStickyDescriptionFitsGitHubLimit` test for description length

Since previous review (e8e26219, addressing a second reviewer's comments):
- Extracted `overrideDescriptionPrefix = "Overridden by"` constant, used by both `description()`/`stickyDescription()` and the cancel-matching logic, so they can't drift independently
- Added explicit parentheses around the baseSHA condition in `tide.go`'s `prowJobsFromContexts` (`config.IsSkipRetest(desc) || (baseSHAForContext != "" && baseSHAForContext == baseSHA)`) — same fix as the precedence nit below, raised independently by Prucek
- PR received `/lgtm` and approval from Prucek, self-approval from author (smg247), and merged 2026-07-14T08:48:24Z

## Review posted

Review comments posted on 2026-07-01T14:16:00Z by petr-muller. Author responded on 2026-07-01T17:24:37Z and pushed a28708fb addressing most feedback. A second reviewer, Prucek, reviewed on 2026-07-13 and raised two nits (constant extraction, parentheses); author addressed both in e8e26219. PR merged 2026-07-14.

## Findings

### [should-fix] Regular /override should reuse baseSHA from the overridden context
- where: `pkg/plugins/override/override.go:513`
- concern: `baseSHAGetter()` fetches the current base branch HEAD at override time. For regular `/override`, this is wrong — it should reuse the baseSHA already embedded in the overridden context's description (via `BaseSHAFromContextDescription`). The override just changes the verdict, it shouldn't claim the job ran against a different base. If the original context has no baseSHA, fall back to the current base as today.
- posted: yes (petr-muller, override.go:513)
- status: OPEN — not addressed in a28708fb

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

### [nit] Operator precedence ambiguity in Tide condition
- resolved in: e8e26219
- how: Added explicit parentheses: `config.IsSkipRetest(desc) || (baseSHAForContext != "" && baseSHAForContext == baseSHA)`. Raised independently by both petr-muller (this review) and Prucek (second reviewer).

### [nit] "Overridden by" string literal duplicated between description() and cancel logic
- resolved in: e8e26219
- how: Extracted `overrideDescriptionPrefix = "Overridden by"` constant, used by `description()`, `stickyDescription()`, and `handleOverrideCancel`'s match check. Raised by second reviewer (Prucek); not in the original findings list.

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

## Followups

Assessed on 2026-10-10. PR #778 merged as `0633879af8026d056e1a5dbe1e29f5a98f6acec3`. Followups were checked against current `main` (`1633048b7`, 2026-10-10), including subsequent override fixes.

Decisions: **2 accepted; 2 skipped because existing work already tracks them.**

- Preserve the original status base SHA: already implemented by open [PR #808](https://github.com/kubernetes-sigs/prow/pull/808), under [issue #789](https://github.com/kubernetes-sigs/prow/issues/789). The PR remains open; this is covered work, not a resolved finding.
- Investigate removing synthetic override ProwJobs: already tracked by [issue #789](https://github.com/kubernetes-sigs/prow/issues/789). No duplicate handoff was accepted.

### [accepted] Test Tide's sticky-status contract

- category: tests
- necessity: should — this contract determines whether Tide requires fresh presubmit results.
- where: `pkg/tide/tide_test.go` (`TestAccumulate`); `pkg/tide/tide.go` (`prowJobsFromContexts`).
- why followup: The merged PR tested override status formatting but added no Tide regression cases for the sentinel. Current main still lacks those cases; the behavior can be covered independently without changing the merged implementation.
- handoff:

```text
In kubernetes-sigs/prow, add Tide regression coverage following PR #778, "Add /override-sticky command for persistent overrides" (merge commit 0633879af8026d056e1a5dbe1e29f5a98f6acec3). Work from current main, incorporating later fixes rather than the original PR branch.

Extend the existing TestAccumulate cases in pkg/tide/tide_test.go to exercise the skip-retest contract consumed by prowJobsFromContexts in pkg/tide/tide.go. Use successful status contexts carrying config.SkipRetestSentinel with (1) a different base SHA and (2) no embedded base SHA. Verify Tide accepts both without a stored ProwJob. Add failed and pending status cases carrying the sentinel and verify the sentinel alone does not supply a passing result. Isolate these cases from existing passing ProwJobs so cached success cannot mask the behavior.

Acceptance: both successful sticky cases pass; failed/pending sentinel cases cannot synthesize success; existing ordinary-status cases retain matching-base acceptance and stale-base rejection. Run go test ./pkg/tide and report the result.

Scope: tests only, using the existing Tide fixtures. Do not alter production behavior, rewrite override tests, or implement the base-SHA fix already proposed in PR #808.
```

### [accepted] Document override commands and Tide interactions

- category: docs
- necessity: should — users need to understand the stale-result protection that sticky overrides bypass.
- where: `site/content/en/docs/components/plugins/override.md` (new); `site/content/en/docs/jobs.md`.
- why followup: The original review requested a plugin guide, but the merged PR only updated generated command help. Current main still has no override plugin page and only briefly mentions `/override` in the jobs guide.
- handoff:

```text
In kubernetes-sigs/prow, document override behavior following PR #778, "Add /override-sticky command for persistent overrides" (merge commit 0633879af8026d056e1a5dbe1e29f5a98f6acec3). Work from current main and describe shipped behavior, including later fixes in PRs #818, #908, and #967.

Create site/content/en/docs/components/plugins/override.md using neighboring plugin-page conventions, and link it from the override mention in site/content/en/docs/jobs.md. Cover enabling the plugin, authorization/configuration, quoted and multiple context names, /override versus /override-sticky, and /override-cancel with a named context or no arguments. Explain base-branch movement, new HEAD commits, explicit reruns, and the distinct behavior of status contexts and GitHub check runs.

Use pkg/plugins/override/override.go, trigger, and Tide on current main as the source of truth. Check what explicit reruns can replace rather than promising that sticky overrides survive every event. PR #808 is the open base-SHA preservation fix: do not describe its behavior as shipped unless it has merged by execution time.

Acceptance: the new page follows site conventions, the jobs guide links to it, command examples match current parsing/help, and lifetime/cancellation claims match status and check-run implementations. Run the repository's documented documentation checks, if available.

Scope: documentation only. Do not change command behavior, implement outstanding fixes, or introduce a new status-description protocol.
```
