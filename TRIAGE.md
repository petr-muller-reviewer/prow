---
issue: kubernetes-sigs/prow#768
title: "Support sticky overrides via /override-sticky command"
state: closed
labels: ""
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-26T23:33:03Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 3
recommended_labels: [kind/feature, area/plugins, area/tide]
---

## Initial validation

**Assessment**: LEGITIMATE (retrospectively — issue is now CLOSED, resolved by a merged PR)

Stephen Goeddel (`smg247`) filed a well-scoped feature request: `/override`-ed jobs get re-triggered by Tide whenever the base branch moves, even for overrides where the job is simply broken/irrelevant rather than "reviewed and will be fixed later." The issue included a concrete proposal (a `sticky_for_head` config option, a `[prow:sticky]` status-description sentinel, new `/override-sticky` and `/override-cancel` commands), linked prior art (#238, closed without resolution), and clear use-case justification. He self-assigned on 2026-06-29 and opened PR #778 implementing it the same day; the PR merged 2026-07-14 and the issue auto-closed.

**Issue Category**: Feature Request (plugins/override, tide)

**Repository Scope Check**:
- Component: `pkg/plugins/override`, `pkg/tide`, `pkg/config` — all in this repo
- Relevant code paths: `pkg/plugins/override/override.go`, `pkg/tide/tide.go:1043-1073`, `pkg/config/config.go:3416-3470`

**Information Completeness**: Sufficient — proposal included config shape, command semantics, and mechanism.

### Recommendation

No action needed on the issue itself (already closed by the merging of PR #778). This re-triage exists to record a review of the *actual* shipped implementation, since it diverges from the issue's original proposal in several respects (see Research findings).

## Research findings

### What actually shipped (PR #778, merged 0633879af8026d056e1a5dbe1e29f5a98f6acec3)

The proposal evolved materially across the PR's 7 commits before merge:

1. **No `sticky_for_head` config gate in the final version.** The issue proposed (and an early PR commit implemented) a per-repo `plugins.Override.StickyForHEAD` boolean that had to be enabled before `/override-sticky`/`/override-cancel` worked. A later commit (`c8a0d6bc3`) removed the gate entirely — `/override-sticky` and `/override-cancel` are unconditionally available wherever the `override` plugin is enabled. Confirmed by fetching the merged blob: no `StickyForHEAD` field remains in `pkg/plugins/config.go`, and `handleGenericComment` always attempts all three regexes (cancel, sticky, plain).
2. **Dependency direction inverted mid-PR.** An early commit made `pkg/tide` import `pkg/plugins/override` to call `override.IsStickyOverride()`. Commit `c8a0d6bc3` ("Move skip-retest contract to config...") reversed this: the sentinel (`[prow:skip-retest]`, renamed from `[prow:sticky]`) and its checker `IsSkipRetest()` now live in `pkg/config/config.go` as a generic, override-agnostic contract. `pkg/tide/tide.go` no longer imports the override plugin at all; `pkg/plugins/override/override.go` imports `config` and calls `config.SkipRetestSentinel` in `stickyDescription()`. This is a materially better design than what the issue/early commits proposed.
3. **Fixed an adjacent latent bug as a side effect.** Regular (non-sticky) `/override` now also calls `config.ContextDescriptionWithBaseSha()` to embed the base SHA in its status description — previously only the plain description text was set, so `config.BaseSHAFromContextDescription()` always returned `""` for overrides and Tide's `prowJobsFromContexts()` never promoted a synthetic ProwJob for them once Sinker reaped the original ProwJob (~`max_prowjob_age`, default 48h). This is exactly the root cause a prior pass of this triage had identified independently ("Override status description lacks baseSHA") — the fix folds it in as a bonus rather than a separate change.
4. **Scope is deliberately narrower than "sticky forever": overrides clear on new push.** `/override-sticky` embeds the sentinel in the status description for the current HEAD SHA only; a new commit produces a new SHA with no statuses, so jobs run normally again. This matches the issue's stated scope ("Not sticky across new commits") and avoids any orphan-cleanup problem.
5. **`/override-cancel [context]`** is new relative to the original override plugin surface — removes sticky (or regular) overrides by flipping the status back to failure, restoring normal retest semantics without requiring a push.

### Key code (as merged)

- `pkg/config/config.go` — `SkipRetestSentinel = "[prow:skip-retest]"`, `IsSkipRetest(description string) bool`. Generic contract, not override-specific.
- `pkg/plugins/override/override.go:44` — `overrideStickyRe`, `overrideCancelRe` regexes alongside existing `overrideRe`.
- `pkg/plugins/override/override.go:332-334` — `stickyDescription()` embeds `config.SkipRetestSentinel`.
- `pkg/plugins/override/override.go:602-` — `handleOverrideCancel()`, restricted to the same authorized-user set as `/override`.
- `pkg/tide/tide.go:1053-1057` — `prowJobsFromContexts()`: `if config.IsSkipRetest(desc) || (baseSHAForContext != "" && baseSHAForContext == baseSHA)`.

### Test coverage

`pkg/plugins/override/override_test.go` was substantially reworked in the same PR: `fakeClient.CreateStatus` no longer special-cases already-`StatusSuccess` contexts (previously it silently dropped updates to already-successful statuses, which would have masked `/override-cancel`'s success→failure transition), and new cases cover `/override-sticky` and `/override-cancel` (single-context and cancel-all). Coverage looks adequate for the shipped surface.

### Historical connection

- #238 (closed 2025-04, "no longer affecting us") — same underlying complaint (Tide re-running overridden jobs), closed without a code fix. #768/PR #778 is the actual resolution of that long-standing gap.

## Effort assessment

**Effort Level**: 3 — Large (requires expertise) — *as executed; recorded for reference since the work is already done.*

### Summary
Touches Tide's core context-accumulation logic (`prowJobsFromContexts`), a config-owned cross-cutting contract, and the override plugin's authorization/command-parsing surface — four files, ~500 lines net across `config.go`, `override.go`, `override_test.go`, `tide.go`. The PR's own commit history shows the author iterating on the architecture (config-gate added then removed, dependency direction reversed) before landing — consistent with a large/expertise-requiring change even though the final diff reads cleanly.

### Factor Analysis
- **Scope**: 4 files, ~500 LOC — Level 2-3
- **Complexity**: Moderate-high — status-description sentinel encoding, Tide's `accumulate()`/`prowJobsFromContexts()` interaction, GitHub status-vs-ProwJob duality — Level 3
- **Required expertise**: Deep — needed working knowledge of both the override plugin and Tide's synthetic-ProwJob promotion path, plus judgment to invert the import direction rather than have Tide depend on a specific plugin — Level 3
- **Clarity/certainty**: High by the time of merge, but the design changed shape mid-flight (sticky_for_head added/removed, sentinel ownership moved) — Level 2-3
- **Testing**: Existing plugin test harness reused; one test-double bug (`CreateStatus` swallowing success→failure transitions) had to be fixed to make the new behavior testable — Level 2-3
- **Backwards compatibility**: Fully additive — new commands, existing `/override` behavior unchanged except now correctly embedding baseSHA (a bug fix, not a behavior change contributors would notice negatively) — Level 1
- **Architectural alignment**: Good fit once the dependency direction was inverted; the final shape (config owns a generic sentinel contract) is more aligned with existing patterns like `ContextDescriptionWithBaseSha` than the initial "Tide imports override" approach — Level 2
- **External dependencies**: None — Level 1

### Recommended labels
- `kind/feature` — new user-facing commands
- `area/plugins` — override plugin
- `area/tide` — Tide accumulation logic touched
- No `good-first-issue`/`help-wanted` — already resolved by the reporter

## Briefing summary

Issue #768 requested sticky overrides so that `/override`-ed jobs marked as "broken/irrelevant" don't get needlessly re-triggered by Tide on every base-branch move. The reporter (`smg247`) self-assigned and shipped it in PR #778 (merged 2026-07-14). The final design differs from the issue's own proposal in two notable ways worth remembering for anyone consulting this record: (1) there's no config flag gating the new commands — `/override-sticky`/`/override-cancel` are always available once the override plugin runs; (2) the sticky-skip-retest contract lives in `config`, not in the override plugin, so Tide doesn't need to import a specific plugin to honor it — a cleaner inversion than the issue's initial framing implied. As a side effect, the PR also fixed a real latent bug where regular (non-sticky) overrides lost their effect after Sinker reaped the synthetic ProwJob (~48h), because the status description never embedded a baseSHA. No further action needed on the issue; it's fully resolved.

## Findings

### [related-pr] PR #778 — Add /override-sticky command for persistent overrides
- merged: 2026-07-14T08:48:24Z, commit `0633879af8026d056e1a5dbe1e29f5a98f6acec3`
- fixes: kubernetes-sigs/prow#768
- files: `pkg/config/config.go`, `pkg/plugins/override/override.go`, `pkg/plugins/override/override_test.go`, `pkg/tide/tide.go`

### [related-code] config.SkipRetestSentinel / IsSkipRetest — generic skip-retest contract
- where: `pkg/config/config.go` (added by PR #778)
- excerpt: |
    const SkipRetestSentinel = "[prow:skip-retest]"
    func IsSkipRetest(description string) bool {
        return strings.Contains(description, SkipRetestSentinel)
    }
- note: deliberately owned by `config`, not `override` — mirrors the existing `ContextDescriptionWithBaseSha`/`BaseSHAFromContextDescription` pattern already in this file, so Tide doesn't need to import the override plugin.

### [related-code] tide.prowJobsFromContexts — now honors the sentinel
- where: `pkg/tide/tide.go:1053-1057`
- excerpt: |
    desc := string(headContext.Description)
    baseSHAForContext := config.BaseSHAFromContextDescription(desc)
    if config.IsSkipRetest(desc) || (baseSHAForContext != "" && baseSHAForContext == baseSHA) {
        passingCurrentContexts = append(passingCurrentContexts, string(headContext.Context))
    }

### [related-code] override.go — /override-sticky, /override-cancel, baseSHA-embedding regular overrides
- where: `pkg/plugins/override/override.go:44` (regexes), `:332-334` (stickyDescription), `:602-` (handleOverrideCancel)
- note: regular `/override` status descriptions now go through `config.ContextDescriptionWithBaseSha(descFn(user), baseSHA)` instead of the bare `description(user)` string used pre-PR — this is the fix for the baseSHA-less-override bug identified in the earlier triage pass.

### [related-issue] #238 — original 2024 report of the same Tide re-run behavior
- ref: kubernetes-sigs/prow#238
- relevance: Closed in 2025 without a code fix ("no longer affecting us"). #768/PR #778 is the actual resolution.

## Checked

- Confirmed via `gh api .../contents/pkg/plugins/override/override.go?ref=<merge-sha>` that the final merged code has no `StickyForHEAD`/`sticky_for_head` config gate — it was added in an intermediate commit and removed before merge.
- Confirmed `pkg/tide/tide.go` no longer imports `pkg/plugins/override` in the final version (only in an intermediate commit) — dependency direction is override → config, not tide → override.
- Confirmed the PR's test changes fix a pre-existing test-double bug (`fakeClient.CreateStatus` previously ignored updates to already-successful statuses), which would have prevented `/override-cancel`'s success→failure transition from being testable.
- Verified issue and PR are fully closed/merged; no open follow-up issues reference #768 or #778 as of this triage.

## Next steps

None — issue is resolved and closed. No labels need to be applied and no comment needs to be posted.

## Open questions

None outstanding. Prior open questions from the initial triage are resolved by the shipped implementation:
- *"Should sticky overrides auto-cancel on new push?"* → Yes, implicitly (new SHA has no statuses).
- *"What happens to sticky labels if the job is removed from required list?"* → Moot — no labels are used; state lives in the per-SHA status description, which naturally becomes irrelevant once the SHA is superseded.
- *"Is label storage sufficient for arbitrary context names?"* → Moot — the shipped design uses status descriptions per context, not a single label enumerating contexts.
- *"Is a Tide-side modification acceptable given the override↔Tide coupling?"* → Resolved by inverting the dependency: Tide depends on a generic `config` contract, not on the override plugin.
