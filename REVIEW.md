---
pr: 615
title: "deck: drop stale non-blocking presubmit contexts from PR status"
head_sha: fe6470203fbd6edee453014dd37e340526281a24
base: main
reviewed_at: "2026-06-09"
verdict: APPROVE WITH SUGGESTIONS
---

# PR #615 — deck: drop stale non-blocking presubmit contexts from PR status

**Author:** Pnkcaht (Sam Richard)
**File:** `cmd/deck/static/pr/pr.ts`
**Diff:** +28 / -33
**Labels:** area/deck, size/M
**Fixes:** test-infra#36399

## What this PR is about

**Problem:** When a presubmit job is renamed or removed from Prow config, its old GitHub status context persists on the PR forever (GitHub never deletes status contexts). Deck's PR status page shows these stale contexts alongside current jobs, confusing PR authors about what is actually required to merge.

**Approach:** Make ProwJobs the sole source of truth for the context list. GitHub status contexts are used only to enrich existing ProwJob entries (detecting state mismatches). Contexts that exist only on GitHub with no corresponding ProwJob are dropped entirely.

## Behavior change

| Before | After |
|--------|-------|
| Start with **all** GitHub contexts | Start with **only** current ProwJobs |
| Overlay ProwJob data on matching contexts | Enrich with GitHub context state for mismatch detection |
| GitHub-only contexts (stale jobs, external CI) **remain visible** | GitHub-only contexts are **silently dropped** |

## Key diff in `getFullPRContext()`

`cmd/deck/static/pr/pr.ts:335–376`

```diff
 function getFullPRContext(builds: ProwJob[], contexts: Context[]): UnifiedContext[] {
   const contextMap: Map<string, UnifiedContext> = new Map();

+  // Build contexts strictly from current ProwJobs
+  for (const build of builds) { ... contextMap.set(context, {...}); }

+  // Enrich with GitHub contexts (no new entries allowed)
+  if (contexts) {
+    for (const ghContext of contexts) {
+      if (ghContext.Context === "tide") continue;
+      if (!contextMap.has(ghContext.Context)) continue;  // <-- the key change
+      // detect state mismatch...
+    }
+  }
   return Array.from(contextMap.values());
 }
```

## Reviewer verdicts

- **Code Quality: APPROVE** — No critical issues. Clean inversion of logic. Pre-existing type safety gaps noted but not regressed.
- **Maintainability: APPROVE** — LOW burden. Net -5 lines, clearer mental model. Missing tests and dropped TODO noted.
- **Deployment Risk: APPROVE** — LOW risk. Client-side only, instant rollback. Non-Prow context loss is the key behavioral concern.

## Converging concerns

### Flagged by all 3 reviewers

- **Drops all GitHub-only contexts, not just stale Prow ones.** The title says "non-blocking presubmit contexts" but the code drops every GitHub context without a matching ProwJob — including GitHub Actions, external CI, and third-party bots. The backend (`pkg/prstatus/prstatus.go:337-373`) fetches both commit statuses and check runs. Installations using Deck as a unified CI dashboard lose visibility into non-Prow checks.

### Flagged by 2 reviewers (Maintainability + Deployment Risk)

- **No unit tests for `getFullPRContext()`.** Pre-existing gap, but this behavioral change makes coverage more important. The function is pure (Map in, array out) and straightforward to test.
- **Scope broader than title suggests.** Title says "non-blocking" but code drops all GitHub-only contexts indiscriminately, including blocking ones.

## Detailed findings

### [CRITICAL] Drops ALL GitHub-only contexts, not just stale Prow ones (all 3 reviewers)

The title says "non-blocking presubmit contexts" but the code drops every GitHub context that lacks a matching ProwJob. This includes contexts from non-Prow CI systems (GitHub Actions, external CI, third-party bots). Deployments using external CI alongside Prow will lose visibility into those contexts in Deck's PR status view.

**Consider:** Only drop contexts whose name matches Prow job naming conventions (e.g. `pull-*`), or explicitly document this as an intended limitation and get sign-off.

### [WARNING] No tests for `getFullPRContext()` (2 reviewers)

This function has no existing tests, and the PR does not add any. This is a meaningful behavioral change that would benefit from test cases covering: ProwJob-only contexts, GitHub enrichment (match + mismatch), GitHub-only context dropping, and `tide` filtering. Can be a follow-up PR.

### [WARNING] Config errors become invisible (Maintainability)

If a ProwJob fails to be created due to a configuration error (not intentional removal), the corresponding GitHub context vanishes from the UI with no trace. This could make debugging configuration issues harder — the operator sees fewer contexts than expected but has no indication why.

### [INFO] TODO comment about state equivalence was dropped (Maintainability)

The original code had a `TODO (qhuynh96)` noting that ProwJob states and GitHub context states are not equivalent in some cases. The mismatch detection is preserved, but the caveat is lost. Consider adding a brief note near the `as UnifiedState` cast at line 367.

### [INFO] Unsafe `as UnifiedState` type assertion (Code Quality, pre-existing)

The cast at line 367 (`ghContext.State.toLowerCase() as UnifiedState`) is a type assertion without runtime validation. If GitHub ever sends an unexpected state value, it would silently pass. This is pre-existing behavior, not a regression.

### [POSITIVE] Clean code quality improvement (all reviewers)

The refactored function is cleaner and easier to follow. The two-phase "ProwJobs are truth, GitHub enriches" pattern creates a clear mental model. Net -5 lines. Variable renaming (`ghContext`, `prowContext`) improves readability. Discrepancy detection and tide filtering are both preserved.

## Existing review comments

**ivankatliarchuk** raised the same scope concern: the PR title says "non-blocking presubmit contexts" but the change drops all GitHub-only contexts, which could affect required status checks from non-Prow CI.

## Suggestions for the author

1. **Consider preserving non-Prow contexts:** If any installation uses Deck as a unified CI dashboard showing GitHub Actions alongside Prow, this change breaks that view. A simple check — only dropping GitHub-only contexts that match Prow naming patterns — would scope the fix more precisely.
2. **Add unit tests for `getFullPRContext()`:** Even a handful of cases covering matching context, orphaned GitHub context, ProwJob without GitHub context, and tide filtering would reduce regression risk. Can be a follow-up PR.
3. **Restore the TODO about state non-equivalence:** Add a brief note near the `as UnifiedState` cast at line 367 so future maintainers know ProwJob/GitHub states are not 1:1 equivalent.
4. **Clarify the PR title:** A more accurate title would be something like "deck: use ProwJobs as source of truth for PR status contexts" since the change affects all context types.

## Questions for the author

- Is dropping *all* non-ProwJob contexts intentional, or should this be scoped to only Prow-style context names?
- Are there known deployments of Prow that rely on non-Prow GitHub contexts appearing in Deck's PR status?
- Would it be feasible to add basic tests for `getFullPRContext()` given the TypeScript test setup in this repo?
- The TODO about ProwJob/GitHub state non-equivalence was removed — has that been resolved, or should the comment be preserved?
- The PR description mentions follow-up work on Prow comment rebuilding — does this PR work correctly in isolation without that?

## Deployment notes

- Client-side only change — no config migration or downtime required. Takes effect on next Deck deployment.
- Rollback is instant: redeploy previous Deck image.
- Operators using Deck's PR status page to view non-Prow CI signals (GitHub Actions, external CI) should be aware those will no longer appear.

## Verdict: APPROVE WITH SUGGESTIONS

All three reviewers approved independently. The change is clean, well-motivated, and correctly solves the stale context problem. The converging concern about non-Prow context visibility is worth discussing but is non-blocking given the practical deployment context of this project.

## Draft PR comment

LGTM. The logic inversion in `getFullPRContext` is clean, well-motivated, and correctly solves the stale non-blocking context problem. No critical issues found.

Two suggestions worth considering: (1) the change drops all GitHub-only contexts, not just stale non-blocking ones — if any installation uses Deck to view non-Prow CI, that visibility is lost; and (2) unit tests for `getFullPRContext` would be valuable given the behavioral change, even as a follow-up.

Approving as-is since these are non-blocking.
