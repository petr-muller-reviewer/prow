---
pr: 579
title: "tide: add configurable GitHub merge blocks enforcement"
head_sha: 2c14b702
base: main
reviewed_at: "2026-05-30T23:47:19Z"
verdict: approve
refresh_log:
  - from: 7f08e035
    to: 2c14b702
    date: "2026-05-30T23:47:19Z"
    summary: "Rev 2 commit addresses 5 of 8 original findings; confirmed item 10 resolved"
gate:
  decision: merge
  gated_at: "2026-05-31T00:22:37Z"
  gated_head_sha: 2c14b702
  reviewed_head_sha: 7f08e035
---

# PR #579 — tide: add configurable GitHub merge blocks enforcement

[kubernetes-sigs/prow#579](https://github.com/kubernetes-sigs/prow/pull/579)
· Author: **Prucek** (Peter Rucek)
· +1097 / -11
· Fixes [#575](https://github.com/kubernetes-sigs/prow/issues/575), [#269](https://github.com/kubernetes-sigs/prow/issues/269)
· Previous attempt: [#537](https://github.com/kubernetes-sigs/prow/pull/537) (reverted in [#574](https://github.com/kubernetes-sigs/prow/pull/574))

## Gate

**Decision: MERGE** (gated at `2c14b702`, 2026-05-31)

All eight inline review comments from the Apr 27 review have been dispositioned: six addressed in rev 2 code changes, one replied to by the author (GitHub reason surfacing — reasonable follow-up), one cosmetic (test comment accuracy — not blocking). No `CHANGES_REQUESTED` reviews on GitHub. No `lgtm` or `approved` labels yet — those are the procedural gate, not a code gate.

**Area 1 — Prior findings disposition:** All `should-fix` items from `REVIEW.md` are addressed in `2c14b702`. The two not-addressed items (misleading test comment #2, warning log level #4) are nits/suggestions — neither blocks merge. The author's reply on item 12 (surface GitHub blocking reason) is a reasonable future follow-up.

**Area 2 — Independent merge risk:**
- **Configuration**: New optional field `github_merge_blocks_policy` with `omitempty`, defaults to `permit` (preserves existing behavior). No existing fields renamed/removed. Backward-compatible. If operators adopt the field and roll back the binary, they must remove it from config first (parse failure).
- **Behavioral**: Under the default `permit` policy, PRs with `mergeStateStatus=BLOCKED` now show `"In merge pool (despite BLOCKED)."` instead of `"In merge pool."`. Operators with status-text-matching monitoring should be aware.
- **API surface**: `MergeStateStatusBlocked` is exported but only used within `tide` package. `GitHubMergeBlocksPolicy` type and constants are exported in `pkg/config` — appropriate for config types. No breaking changes to existing exported API.
- **GraphQL**: Adds `mergeStateStatus` field to the PR query — lightweight, no additional API call.

**Gating list:** No items gate merge. All prior blocking/should-fix findings are addressed. No backward-incompatible changes.

## Revision tracking

**Rev 2** — Commit `2c14b702` (Apr 30) — addresses 6 of 8 review comments from `7f08e035`. Prucek replied to 1 comment (GitHub reason surfacing). 2 comments still open. Refreshed May 30.

## Reviewer perspectives — Gate check

### Code Quality
- **APPROVE**
- No critical issues
- Correct logic, safe, idiomatic Go
- Comprehensive test coverage
- Exported constant could be unexported
- Misleading test comment (cosmetic)

### Maintainability
- **APPROVE** · Burden: LOW
- Follows established patterns exactly
- Method/type name collision (minor)
- poolStatus warning log could be noisy
- requirementDiff mixes two concerns

### Deployment Risk
- **Risk: LOW**
- No breaking changes
- Safe default preserves behavior
- Status string change visible on upgrade
- New warning logs under default policy

## Files changed (8)

- M `pkg/config/config.go` (+8)
- M `pkg/config/prow-config-documented.yaml` (+7)
- M `pkg/config/tide.go` (+41)
- M `pkg/tide/github.go` (+12)
- M `pkg/tide/status.go` (+31/-6)
- M `pkg/tide/status_test.go` (+811)
- M `pkg/tide/tide.go` (+8/-5)
- M `pkg/tide/tide_test.go` (+179)

## Positive aspects

- Clean three-tier policy design (`ignore`/`permit`/`block`) follows existing Tide config patterns like `PrioritizeExistingBatchesMap` and `BatchSizeLimitMap`
- Safe default: `permit` preserves existing behavior — the right call for a previously-reverted feature
- Thorough testing: ~990 lines of new tests covering `requirementDiff`, config resolution hierarchy, `poolStatus`, `isAllowedToMerge`, and priority ordering
- Config validation at load time rejects invalid policy values (`parseProwConfig`)
- Defensive `default:` case in switch returns error for unexpected policy values
- Correct GraphQL field mapping: `mergeStateStatus` tag matches GitHub's API
- Proper org/repo config resolution order: repo > org > `*` > default
- `requirementDiff` and `poolStatus` now receive just the policy, not the full config (clean interface)
- `MergeStateStatusBlocked` constant replaces raw `"BLOCKED"` string literals
- Backwards-compatible: `omitempty` tag, no config migration needed, safely rollback-able

## Review items (12)

### 1. No config validation for policy values [addressed in rev 2]

**Severity:** addressed

**Original concern:** No validation that `github_merge_blocks_policy` values are one of `ignore`, `permit`, or `block`. A typo like `"blocker"` would silently fall through.

**Rev 2:** Validation added in `parseProwConfig` (`config.go:2780-2786`) — iterates the map and returns an error for any value not matching the three constants. The `default:` case in `isAllowedToMerge` (`github.go:644`) now returns `fmt.Errorf` as a defense-in-depth.

**Remaining note:** No separate `checkconfig` validation, but `checkconfig` calls `Load()` which runs `parseProwConfig`, so invalid values are caught.

`pkg/config/config.go:2780-2786`, `pkg/tide/github.go:644`
*Originally flagged by: Code Quality, Maintainability, Deployment Risk*

### 2. Misleading test comment and test name [2/3 reviewers]

**Severity:** nit

The test case "Complex: All requirements met but changes requested blocks merge" at `status_test.go:668-669` has a comment claiming `+ 50(review)` but `reviewApprovedRequired` is `false`, so the review check contributes 0. The `expectedDiff: 100` is correct but the comment is wrong.

Also, `TestIsAllowedToMerge_ReviewDecision` in `tide_test.go` actually tests `MergeStateStatus`/`GitHubMergeBlocksPolicy`, not review decisions.

**Rev 2:** Not addressed. Non-blocking — cosmetic accuracy issues in test comments and naming.

`pkg/tide/status_test.go:668-669`, `pkg/tide/tide_test.go:1523`
*Flagged by: Code Quality, Maintainability*

### 3. Raw "BLOCKED" string compared in three places [addressed in rev 2]

**Severity:** addressed

**Original concern:** The `MergeStateStatus` field was compared against the raw string `"BLOCKED"` in three separate locations.

**Rev 2:** `MergeStateStatusBlocked = "BLOCKED"` constant added at `tide.go:1911`, near the `PullRequest` type. Used consistently in all three locations.

**Minor note:** The constant is exported but only used within the `tide` package. Could be unexported (`mergeStateStatusBlocked`). Non-blocking.

`pkg/tide/tide.go:1911`
*Originally flagged by: Maintainability*

### 4. Warning log fires every sync cycle per BLOCKED PR

**Severity:** suggestion

Under the default `"permit"` policy, `poolStatus()` emits a warning log for every BLOCKED PR on every status sync cycle. For installations where many PRs commonly have BLOCKED status, this could produce log noise.

**Mitigating factor:** Under `"permit"`, the PR merges normally, so the noisy window is just the sync cycles between entering the pool and merging. Unlikely to be a real problem in practice.

**Rev 2:** Not addressed. Consider downgrading to `Info` or `Debug` level in a follow-up.

`pkg/tide/status.go:374-386`
*Flagged by: Code Quality, Maintainability, Deployment Risk*

### 5. Dual enforcement points lack cross-references

**Severity:** nit

The BLOCKED state check with policy resolution appears in both `isAllowedToMerge()` (merge gating) and `requirementDiff()` (status reporting). These serve different purposes and are functionally correct.

**Rev 2:** Partially improved — now that `requirementDiff` takes the policy directly (not the full config), the two enforcement points are more clearly independent. A cross-reference comment would still help future maintainers.

`pkg/tide/github.go:635`, `pkg/tide/status.go:264`
*Flagged by: Maintainability, Deployment Risk*

### 6. BLOCKED diff weight same as milestone mismatch (100)

**Severity:** question

In `requirementDiff`, the BLOCKED state adds `diff += 100`, which is the same weight as a milestone mismatch. Since BLOCKED represents a hard GitHub-side enforcement, should it have a higher weight?

If both milestone and BLOCKED are wrong, the milestone description wins because it's checked first. The user would see "Must be in milestone X" instead of the more actionable "Blocked by GitHub" message.

`pkg/tide/status.go:264` (diff += 100), `pkg/tide/status.go:186` (milestone also diff += 100)

### 7. Pass just the policy, not the full config [addressed in rev 2]

**Severity:** addressed

**Original concern:** Both `requirementDiff` and `poolStatus` received `*config.Tide` but only needed the resolved policy.

**Rev 2:** Both functions now take `config.GitHubMergeBlocksPolicy` directly. The policy is computed once in `expectedStatus` (`status.go:288`) and passed to both `requirementDiff` and `poolStatus`. Clean separation.

`pkg/tide/status.go:129`, `pkg/tide/status.go:288`, `pkg/tide/status.go:374`
*Originally flagged by: Code Quality, Maintainability*

### 8. Status string change may affect monitoring

**Severity:** deploy note

PRs in the merge pool with BLOCKED merge state under the default `"permit"` policy will now show `"In merge pool (despite BLOCKED)."` instead of `"In merge pool."` in the GitHub status context.

Any automation or monitoring that matches on the exact string `"In merge pool."` will see a different value for these PRs. Should be mentioned in release notes.

`pkg/tide/status.go:50`
*Flagged by: Deployment Risk*

### 9. poolStatus logging: hardcoded policy value [improved by item 7]

**Severity:** nit

In `poolStatus`, the logrus field `"policy"` is hardcoded to `config.GitHubMergeBlocksPermit` rather than referencing the parameter.

**Rev 2:** `poolStatus` now receives the policy as a parameter (`mergeBlocksPolicy`), but the log field still hardcodes `config.GitHubMergeBlocksPermit` instead of using the parameter. Technically correct since the code path only executes when policy is permit, but using the parameter would be more natural. Very minor.

`pkg/tide/status.go:381`
*Flagged by: Code Quality*

### 10. Misleading "Block merge" comment in status controller [addressed in rev 2]

**Severity:** addressed

**PR comment posted:** `status.go` — "this is misleading, this is status controller - does not actually block merges"

The status controller's `requirementDiff` adds to `diff` which affects the status message, not the merge decision. The actual merge blocking happens in `isAllowedToMerge()` in `github.go`.

**Rev 2:** Confirmed addressed. The diff shows the misleading "Block merge by adding to diff" comment and surrounding explanatory comments were removed. The code at `status.go:264-269` is now a bare condition (`if mergeBlocksPolicy == config.GitHubMergeBlocksBlock && pr.MergeStateStatus == MergeStateStatusBlocked`) with no misleading commentary.

`pkg/tide/status.go:264-269`
*Found during review*

### 11. Config documentation clarity [partially addressed]

**Severity:** suggestion

**PR comment posted:** "What kind of issues may `ignore` cause? 'Restrict updates' in what sense?"

**Rev 2:** The doc comment was rewritten and is now clearer — explains what the field does, mentions `mergeStateStatus` and BLOCKED state, lists valid values, and notes the default. The confusing "restrict updates" phrasing is gone.

**Remaining gap:** The comment still doesn't explain what issues `ignore` may cause (e.g., that Tide may attempt merges that GitHub will reject, leading to silent failures like #269). Could be addressed in a follow-up doc improvement.

`pkg/config/prow-config-documented.yaml:1563-1567`
*Found during review*

### 12. Surface actual GitHub blocking reason (future) [author replied]

**Severity:** suggestion

When Tide reports "Blocked by GitHub", it would be more helpful to show the actual reason. GitHub provides informative messages on denied merge attempts:

```
At least 2 approving reviews are required by reviewers with write access.
```

**Prucek's reply (Apr 30):** "I think there is no API to get it before the merge. We can maybe add the status after it tries to merge, from the mergeError."

Reasonable follow-up for a future PR. Not a blocker.

`pkg/tide/status.go:267`
*Found during review*

## Verdict

**APPROVE**

The implementation is clean, well-tested, and backward-compatible. Rev 2 addressed the key review comments: config validation, BLOCKED constant extraction, passing policy instead of full config, and defensive default case in switch. The remaining items (test comment accuracy, log level, doc clarification) are non-blocking and can be handled in follow-ups. Gate check passes with no blocking issues.

## Followups

### 1. Surface actual GitHub blocking reason in Tide status

**Category:** deferred-review | **Necessity:** should | **Where:** `pkg/tide/status.go:267`, `pkg/tide/github.go:635-646`

```
In kubernetes-sigs/prow, following PR #579 ("tide: add configurable GitHub merge
blocks enforcement"), Tide now reports "Blocked by GitHub (branch rulesets or
protection)" when a PR has mergeStateStatus=BLOCKED. This is a generic message —
it doesn't tell the operator which specific protection rule fired.

GitHub provides informative messages when a merge attempt is actually rejected,
e.g. "At least 2 approving reviews are required by reviewers with write access."
The PR author (Prucek) noted there may be no pre-merge API for this, but
suggested surfacing the reason from the merge error after a failed attempt.

Task:
1. Investigate whether GitHub's GraphQL API exposes the specific blocking reason
   before a merge attempt — check the `mergeStateStatus` field's siblings on the
   PullRequest type, and any `mergeRequirements` or `branchProtectionRule` fields.
2. If a pre-merge field exists: read it in `isAllowedToMerge` (github.go:635)
   and `requirementDiff` (status.go:264) and include the reason in the status
   description and the merge-blocked message.
3. If no pre-merge field exists: in the merge path (where Tide actually attempts
   the merge and gets an error), capture the error message from GitHub's response
   and surface it — either by updating the PR status context after a failed merge,
   or by logging it with structured fields for operator visibility.
4. Update tests in status_test.go and tide_test.go to cover the new reason text.

Acceptance criteria:
- When a PR is blocked, the Tide status or log shows the specific GitHub reason
  (e.g. "required reviews", "status checks") instead of the generic message.
- Existing tests pass; new tests cover the reason-surfacing path.
- If the pre-merge API doesn't exist, document that in a code comment and
  implement the post-merge-attempt fallback.

Out of scope:
- Changing the three-tier policy design (ignore/permit/block).
- Changing the default policy.
- Any config schema changes.
```
