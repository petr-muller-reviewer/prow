---
pr: kubernetes-sigs/prow#964
title: "deck: fix job bar state proportions"
head_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
base: main
reviewed_at: 2026-09-23T21:11:45Z
verdict: request-changes
refresh_log:
  - old_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
    new_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
    at: 2026-09-23T21:11:45Z
    summary: "No code changes; incorporated the author's dashboard screenshot comment."
---

## What this PR does

- Normalizes missing and unrecognized states into the job bar's `unknown` segment.
- Gives supported job states, including `scheduling`, fixed job-bar segments.
- Replaces the previous order-dependent final `auto` width with per-state widths.

Since previous review:

- Prucek added a dashboard screenshot showing the missing scheduling segment and distorted unknown proportions; no commit, inline review comment, or submitted review followed.

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
