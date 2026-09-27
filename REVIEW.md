---
pr: kubernetes-sigs/prow#964
title: "deck: fix job bar state proportions"
head_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
base: main
reviewed_at: 2026-09-27T15:32:03Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-09-27T15:32:03Z
  gated_head_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
  reviewed_head_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
refresh_log:
  - old_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
    new_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
    at: 2026-09-23T21:11:45Z
    summary: "No code changes; incorporated the author's dashboard screenshot comment."
  - old_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
    new_sha: 7f6976fe8e75fc87a3fdbc52ffb5144fc7ba9c2c
    at: 2026-09-27T15:32:03Z
    summary: "No code changes; accepted minor width distortion from the 1% minimum and downgraded the coverage gap."
---

## Gate

**Decision: merge.** The PR is still at the reviewed head. The 1% floor can slightly distort sparse-state widths, but this is acceptable under the stated goal of avoiding a badly distorted job bar. The change fixes the missing scheduling segment and aggregates unknown states into a dedicated segment. The coverage gap does not establish a functional regression.

Gating findings:

- None under the accepted tolerance for small proportional differences. The previous 1% floor concern is an accepted display tradeoff, and focused test coverage remains a non-gating improvement.

Independent merge risk:

- The four-file PR diff changes Deck's default job-bar display for every existing Deck deployment: unknown and future states are aggregated into `unknown`, scheduling gets its own segment, and all segments receive explicit percentage widths (`cmd/deck/static/prow/prow.ts:577-581,800-831`; `cmd/deck/template/index.html:42-43`). The 1% floor may slightly overstate sparse states; this is accepted and does not create an unacceptable merge risk.
- No backend or Kubernetes API, configuration, wire format, or migration change appears in the diff. `cmd/deck/static/api/prow.ts:2` only widens the frontend TypeScript state union to include the existing `scheduling` value.

## What this PR does

- Normalizes missing and unrecognized states into the job bar's `unknown` segment.
- Gives supported job states, including `scheduling`, fixed job-bar segments.
- Replaces the previous order-dependent final `auto` width with per-state widths.

Since previous review:

- Prucek added a dashboard screenshot showing the missing scheduling segment and distorted unknown proportions; no commit, inline review comment, or submitted review followed.
- The 1% minimum was accepted as a display tradeoff; no code changes followed.

## Findings

No actionable findings under the accepted display tolerance.

## Checked

- State normalization retains known states, including `scheduling`, and aggregates missing or unrecognized values under `unknown`.
- A fixed state order eliminates the previous map-order-dependent final `auto` segment and resets absent segments to zero width.
- The change is static Deck TypeScript/HTML/CSS only: no ProwJob API, configuration, RBAC, or storage migration changes.
- `scheduling` matches the existing backend state, so no producer-side rollout is needed.
- The 1% minimum can make sparse segments wider than their exact share, but there are at most eight state segments. This is an accepted display tradeoff rather than a merge blocker.
- There is no focused job-bar test for normalization, sizing, or redraws; adding one would protect the behavior against future regressions.

## Open questions

- Should this change also add `scheduling` to the shared table-state icon and color handling in `cmd/deck/static/common/common.ts` and `cmd/deck/static/style.css`? It currently receives bar treatment but no table icon.

## Followups

### [tests; should] Cover Deck job-bar states and widths

```text
In kubernetes-sigs/prow, following PR #964 "deck: fix job bar state proportions", add focused automated coverage for Deck's job-bar state normalization and segment sizing in cmd/deck/static/prow/prow.ts. Verify that scheduling remains distinct, empty and unrecognized states aggregate under unknown, absent states reset to zero width on redraw, and nonzero segments follow the accepted 1% minimum. Use the existing Deck frontend test setup (see cmd/deck/static/prow/histogram_test.ts); extract only the smallest helper needed for a focused test. Acceptance criteria: the new tests exercise those cases and the relevant Deck frontend test command passes. Keep the current job-bar appearance and API unchanged; do not alter the accepted width floor or expand into a broader UI rewrite.
```

### [UI consistency; could] Show scheduling in Deck job rows

```text
In kubernetes-sigs/prow, following PR #964 "deck: fix job bar state proportions", update the shared Deck state-cell presentation in cmd/deck/static/common/common.ts (cell.state, around lines 52-89) and cmd/deck/static/style.css (state colors, around lines 159-181) so a scheduling ProwJob has a visible icon and appropriate state color, consistent with the new scheduling job-bar segment. Acceptance criteria: a scheduling job row shows a recognizable icon and readable color in light and dark modes, while existing state rows keep their current presentation. Add a focused test if the existing frontend test setup supports the state cell. Do not change job-bar sizing or filtering, backend state definitions, or unrelated UI components.
```
