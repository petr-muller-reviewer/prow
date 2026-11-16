---
pr: kubernetes-sigs/prow#964
title: "deck: fix job bar state proportions"
head_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
base: main
reviewed_at: 2026-11-16T23:48:13Z
verdict: request-changes
refresh_log:
  - old_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
    new_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
    at: 2026-09-23T21:11:45Z
    summary: "No code changes; incorporated the author's dashboard screenshot comment."
  - old_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
    new_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
    at: 2026-11-16T23:48:13Z
    summary: "No code changes; incorporated petr-muller's approval and the PR being merged with both findings open."
---

## What this PR does

- Normalizes missing and unrecognized states into the job bar's `unknown` segment.
- Gives supported job states, including `scheduling`, fixed job-bar segments.
- Replaces the previous order-dependent final `auto` width with per-state widths.

Since previous review:

- Prucek added a dashboard screenshot showing the missing scheduling segment and distorted unknown proportions; no commit, inline review comment, or submitted review followed.
- petr-muller approved the PR at 2026-09-27T15:32:29Z with no inline comments; the kubernetes-prow approval-notifier bot echoed the approval at 15:32:37Z.
- No commits followed the approval: the head stayed at `7f6976fe`, and the PR merged at 2026-09-27T15:56:38Z with both findings unaddressed.

## Findings

### [blocking] Sparse job-bar segments are not proportional
- where: `cmd/deck/static/prow/prow.ts:827-830`
- concern: `Math.max(count / total * 100, 1)` makes every nonzero state smaller than 1% occupy 1% of the bar. With multiple sparse states, segment widths can exceed 100%, contradicting the PR's actual-count-percentage goal and making the dashboard distribution inaccurate. Remove the clamp, or use a separate affordance for tiny counts that does not alter bar widths.
- excerpt: |
    el.textContent = count.toString();
    tt.textContent = \`${count} ${stateToAdj(state)} jobs\`;
    el.style.width = \`${Math.max((count / total * 100), 1)  }%\`;

### [should-fix] Cover job-bar state normalization and sizing
- where: `cmd/deck/static/prow/prow.ts:800-832`
- concern: The changed behavior has no focused test for scheduling, empty or future states aggregating into unknown, sub-1% counts, and redraws. Add coverage so a future state-list or width change cannot silently reintroduce misleading bars.
- excerpt: |
    const jobBarStates: ProwJobState[] = ["success", "pending", "scheduling", "triggered", "error", "failure", "aborted", "unknown"];

    function normalizeJobBarState(state: string): ProwJobState {
      switch (state) {
        case "scheduling":
        case "success":
        // ...
        default:
          return "unknown";
      }
    }

## Checked

- State normalization retains known states, including `scheduling`, and aggregates missing or unrecognized values under `unknown`.
- A fixed state order eliminates the previous map-order-dependent final `auto` segment and resets absent segments to zero width.
- The change is static Deck TypeScript/HTML/CSS only: no ProwJob API, configuration, RBAC, or storage migration changes.
- `scheduling` matches the existing backend state, so no producer-side rollout is needed.

## Open questions

- Is the 1% minimum intended as a discoverability affordance? If so, can we retain that cue without claiming it is proportional or consuming additional bar width?
- Should this change also add `scheduling` to the shared table-state icon and color handling in `cmd/deck/static/common/common.ts` and `cmd/deck/static/style.css`? It currently receives bar treatment but no table icon.


## Followups

Post-merge followups for PR #964, merged as `1253900225d5cb9ada83c6575403e389adb643c1` (head `7f6976fe`). Accepted 4 of 4 candidates on 2026-11-16; item 2 is a probe (`Mode: probe`). The first two items are the review findings that merged unfixed.

### [must] Remove the 1% floor on job-bar segment widths
- category: deferred-review
- mode: deliver
- where: `cmd/deck/static/prow/prow.ts:828-831`
- why: Unfixed blocking finding from the review. `Math.max(count / total * 100, 1)` inflates every sub-1% state, so several sparse states make widths exceed 100% and the bar no longer shows actual proportions.
- prompt: |
    In `kubernetes-sigs/prow`, following PR #964 "deck: fix job bar state proportions" (merged as `1253900225d5cb9ada83c6575403e389adb643c1`), fix the segment width calculation in `drawJobBar` in `cmd/deck/static/prow/prow.ts` (around line 830). The code currently does `el.style.width = \`${Math.max((count / total * 100), 1)}%\``, which floors every nonzero state at 1%; with multiple sparse states the widths sum past 100%, contradicting the PR's goal of rendering actual count percentages. Use the true percentage (`count / total * 100`) and keep the existing `width: 0` / empty text behavior for absent or zero states. If sub-1% segments need a visibility affordance, implement it without changing segment widths (for example, a tooltip, a legend, or `min-width` on a child element) and do not reintroduce a floor on the width itself.

    Acceptance criteria:
    - With `total > 0`, the computed widths of all rendered states sum to 100% within rounding; no combination of sparse states exceeds 100%.
    - A single scheduling job among 1000 renders a 0.1% segment rather than 1%, while the segment text and tooltip still show `1`.
    - Absent states remain `width: 0` with empty text, and clicking a segment still applies the state filter.
    - The TypeScript compiles (`npx tsc -p cmd/deck/static/prow/tsconfig.json --noEmit` after installing dependencies) and the deck static bundle builds with the repo's existing tooling.

    Scope guard: do not change `normalizeJobBarState` or state grouping, the job table, or unrelated deck pages. Do not add unit tests (separate followup) and do not modify Go code.

### [should] Probe job-bar normalization and sizing with throwaway tests
- category: tests
- mode: probe
- where: `cmd/deck/static/prow/prow.ts:800-832`
- why: Unfixed should-fix finding. No test exercises state normalization, unknown aggregation, or segment widths; the probe exists to find real defects, not to add permanent coverage.
- prompt: |
    In `kubernetes-sigs/prow`, following PR #964 "deck: fix job bar state proportions" (merged as `1253900225d5cb9ada83c6575403e389adb643c1`), probe the new job-bar logic in `cmd/deck/static/prow/prow.ts`: `jobBarStates` and `normalizeJobBarState` (~lines 800-815) and `drawJobBar` (~lines 818-832). This is a bug hunt: write throwaway tests, run them, triage failures, and submit only a minimal fix if a real bug is found.

    Exercise at least:
    - `scheduling` stays its own segment rather than folding into `unknown` or `triggered`.
    - `""`, `undefined`, and a future/unrecognized state all aggregate under `unknown`.
    - States absent from the map render `width: 0`, empty text, and empty tooltip.
    - Multiple distinct sub-1% states (for example, 1 job of 1000 each in scheduling, error, and aborted) - inspect the computed widths and whether they exceed the bar's 100%.
    - `total === 0` and an empty map do not render negative or `NaN` widths.
    - Counts from all states reach the correct segment, including the normalization call site in `redraw` (~line 580).

    The helpers are module-private and `drawJobBar` touches the DOM. Export or extract only the minimum needed for the probe (temporary changes are fine), or drive `drawJobBar` against a minimal DOM shim. No jasmine runner is currently wired for this package; compile with `npx tsc` and run the specs with `npx jasmine`, or use a small node harness.

    Acceptance criteria: every behavior above is exercised; every failure is classified as a real implementation bug or a wrong expectation with evidence. If a real bug exists, apply only the minimal fix and report the failing case; if not, finish with no files changed.

    Scope guard: the known 1% width floor is a separate followup - do not fix it and do not report it as a new finding. Delete probe tests and temporary exports before finishing. Do not refactor for testability, restyle, or fix unrelated or pre-existing issues.

### [should] Give `scheduling` rows an icon and state color
- category: tech-debt
- mode: deliver
- where: `cmd/deck/static/common/common.ts:7-14,60-86`; `cmd/deck/static/style.css:159-184`
- why: The PR added `scheduling` to the bar and the API type but not to the shared table-state rendering, so scheduling rows show a blank icon and the default color.
- prompt: |
    In `kubernetes-sigs/prow`, following PR #964 "deck: fix job bar state proportions" (merged as `1253900225d5cb9ada83c6575403e389adb643c1`), add first-class `scheduling` rendering to the shared deck state cell. `ProwJobState` now includes `"scheduling"`, but `State` in `cmd/deck/static/common/common.ts` (lines 7-14) and the icon switch in `cell.state()` (lines ~60-86) do not, so `cell.state("scheduling")` renders an empty `<i>` and no state color. Add `SCHEDULING = 'scheduling'` to the enum, map it to a distinct Material icon in the switch (for example `hourglass_empty`; `schedule` is already used for `triggered`), and add a `.state.scheduling` rule in `cmd/deck/static/style.css` near the other `i.state` color rules using `#03A9F4` to match `#job-bar-scheduling`.

    Acceptance criteria:
    - `cell.state("scheduling")` yields `<i class="material-icons state scheduling">` with a non-empty icon and title `Scheduling`.
    - The jobs table (`cmd/deck/static/prow/prow.ts:616`) and the tide-history table (`cmd/deck/static/tide-history/tide-history.ts:252`) compile, and other states render unchanged.
    - `.state.scheduling` uses the same blue as the job bar; dark mode and other state rules are untouched.
    - TypeScript compiles with the deck static build.

    Scope guard: do not touch the job-bar rendering, the API type union, Go code, or add new states.

### [could] Single-source the job-bar state list and template order
- category: cleanup
- mode: deliver
- where: `cmd/deck/static/prow/prow.ts:800-815`; `cmd/deck/template/index.html:37-54`
- why: Adding `scheduling` required five synchronized edits. The `normalizeJobBarState` switch and the hardcoded template order can drift silently from `jobBarStates`.
- prompt: |
    In `kubernetes-sigs/prow`, following PR #964 "deck: fix job bar state proportions" (merged as `1253900225d5cb9ada83c6575403e389adb643c1`), remove the duplicated state list in the Deck job bar. `jobBarStates` (`cmd/deck/static/prow/prow.ts:800`) duplicates the cases in `normalizeJobBarState` (lines 800-815), and the bar element order is duplicated again by the hardcoded `#job-bar-*` divs and tooltips in `cmd/deck/template/index.html:37-54`. A state missing from the switch silently collapses into `unknown`; a template order mismatch silently reorders the bar.

    Make `jobBarStates` the single source: derive normalization from it (for example `jobBarStates.includes(state as ProwJobState)` or a `Set`) and drop the parallel switch cases while preserving `unknown` for missing or unrecognized values. For the template, either generate the bar div and tooltip pairs from `jobBarStates` at runtime or add an explicit startup assertion that every list entry has a matching `.job-bar-state` element. Keep the visible order and behavior unchanged.

    Acceptance criteria:
    - Adding or removing a supported state requires editing the list (plus CSS) but not a second switch; normalization returns `unknown` for every value not in the list.
    - Rendered bar order, widths, labels, and tooltips are identical for the existing eight states; `unknown` handling is unchanged.
    - TypeScript compiles and the deck static bundle builds.

    Scope guard: do not change width calculation, CSS colors, state filtering, or the shared `common.ts` state rendering. Keep the change local to the job-bar code and template; do not introduce a new shared module.
