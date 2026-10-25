---
pr: kubernetes-sigs/prow#787
title: "rifle: smart reviewer selection; alternative to blunderbuss"
head_sha: 76856c96e99c09c35cafce2e1998d6781db5da88
base: main
reviewed_at: 2026-08-25T23:12:00Z
verdict: approve
refresh_log:
  - from: 8d69f902bcd18cfaf02cb90c26b097286278cdd4
    to: e67069f07a0361184518631dec1321bd5599e0f3
    summary: "2 new commits fix handleStatus err shadowing/return-nil in both plugins, improve validateMutuallyExclusivePlugins to handle org/repo conflicts and ExcludedRepos, add tests, improve blame_test.go. Infinite loop bug still open."
  - from: e67069f07a0361184518631dec1321bd5599e0f3
    to: 6bb540ddd3b64c06b752ee2d10aa43d0f6dd89cd
    summary: "Fixed the infinite-loop bug (blocking finding) by adding else{break} to both GetReviewers loops in reviewer.go. Added documentation for rifle to the reviewers/approvers doc. PR self-approved by author. No remaining blocking findings."
  - from: 6bb540ddd3b64c06b752ee2d10aa43d0f6dd89cd
    to: 6bb540ddd3b64c06b752ee2d10aa43d0f6dd89cd
    summary: "No code change. New inline review comment from Prucek (2026-08-21) flags that GetBlame uses pr.Base.Ref (current base tip) while diff hunk line numbers are relative to the merge-base, causing potential line misalignment when base has advanced. Added as a new should-fix finding; verdict downgraded to needs-discussion pending author response."
  - from: 6bb540ddd3b64c06b752ee2d10aa43d0f6dd89cd
    to: 76856c96e99c09c35cafce2e1998d6781db5da88
    summary: "Fixed the merge-base finding: added GetMergeBase (REST compare API) to the GitHub client and fakegithub, and rifle now resolves the PR's merge-base SHA and blames at that ref instead of pr.Base.Ref, falling back to the base ref if the lookup errors or returns empty. Added client_test.go and rifle_test.go coverage including the fallback paths. Author (smg247) acknowledged the finding inline. Prucek /lgtm'd. PR merged."

## Summary

Adds a new `rifle` plugin that selects PR reviewers using git blame scoring instead of random selection. Extracts shared reviewer logic from blunderbuss into `pkg/reviewer/reviewer.go`. Adds `GetBlame` GraphQL method to the GitHub client. Adds mutual-exclusion validation preventing both plugins on the same repo.

Since previous review (2026-07-01):
- Fixed `handleStatus` err shadowing and `return nil` vs `continue` in both rifle and blunderbuss, using `utilerrors.NewAggregate` to collect errors.
- Improved `validateMutuallyExclusivePlugins` to detect org-vs-repo conflicts and respect `ExcludedRepos`. Added 6 test cases.
- Improved `TestReviewerScorer` to actually call `scorer.scoreReviewers()` instead of duplicating scoring logic inline.
- Fixed import ordering, added `BlameData` field to `FakeClient`.

Since previous review (2026-07-03):
- Fixed the infinite-loop bug in `GetReviewers` (`pkg/reviewer/reviewer.go:121-136`) by adding `else { break }` to both selector loops.
- Added documentation for the rifle plugin to `site/content/en/docs/components/plugins/approve/approvers/_index.md`, including algorithm description and API quota note.

Since previous review (2026-08-21): no code changes; new inline review comment surfaces a potential blame/diff line-number misalignment (see findings).

Since previous review (2026-08-25): merge-base finding fixed; PR merged (see Resolved).

## Findings

### [should-fix] findFallbackReviewers bypasses UseStatusAvailability
- where: `pkg/plugins/rifle/rifle.go:411-460`
- concern: Primary selection respects `IsUserBusy`, but `findFallbackReviewers` picks from `AllOwners()` by blame score or randomly without any availability check. A user correctly skipped as busy during initial selection can be assigned via fallback. Novel to rifle — blunderbuss has no all-owners fallback.
- excerpt: |
    // No IsUserBusy call anywhere in findFallbackReviewers
    for i := 0; i < len(candidates) && len(result) < needed; i++ {
        result = append(result, candidates[i].login)
    }

### [should-fix] Massive code duplication between rifle and blunderbuss
- where: `pkg/plugins/rifle/rifle.go:89-353`, `pkg/plugins/config.go:228-259`
- concern: `handlePullRequest`, `handleGenericComment`, `handleStatus`, and `handle` in rifle are near-identical copies of blunderbuss (only config type and selector differ). The `Rifle` config struct is identical to `Blunderbuss`. `validateRifle` is identical to `validateBlunderbuss`. The handleStatus fix was applied identically to both plugins — a concrete example of the duplication cost.

### [nit] Hand-rolled insertion sort in selectBestReviewer
- where: `pkg/plugins/rifle/rifle.go:380-384`
- concern: Manual insertion sort after shuffle. `sort.SliceStable` after shuffle achieves the same stable-sort-with-random-tiebreaking in one line.
- excerpt: |
    for i := 1; i < len(ranked); i++ {
        for j := i; j > 0 && ranked[j].score > ranked[j-1].score; j-- {
            ranked[j], ranked[j-1] = ranked[j-1], ranked[j]
        }
    }

## Resolved

### [should-fix] GetBlame ref may not match the base the diff hunks were computed against
- resolved_in: `76856c96e`
- resolution: Added `GetMergeBase(org, repo, base, head string) (string, error)` to `pkg/github/client.go` (uses the REST compare-two-commits API) and to `fakegithub.FakeClient` (returns configurable `MergeBaseSHA`). `handle` in `pkg/plugins/rifle/rifle.go:276-286` now resolves `pr.Base.Ref`/`pr.Head.SHA`'s merge-base and blames at that SHA instead of the base branch tip, logging a warning and falling back to `pr.Base.Ref` if the lookup errors or returns empty. `rifle_test.go` adds `TestHandleRifleBlamesMergeBase` covering the success, error, and empty-merge-base fallback paths, plus asserts on `blameRefs` in the existing blame-scoring test. Author (smg247) acknowledged: "this would definitely have led to incorrect results, especially as PRs age." Reviewer (Prucek) `/lgtm`'d after the fix; PR merged.

### [blocking] Infinite loop when all candidates are busy
- resolved_in: `944dde12e`
- resolution: `GetReviewers` (`pkg/reviewer/reviewer.go:121-136`) now breaks out of both selector loops when `selector()` returns `""`, instead of spinning while `unusedLeaves.Len() > 0`/`fileReviewers.Len() > 0` stays true. This was independently flagged by Prucek in an inline review comment on 2026-07-03.
- excerpt: |
    for reviewers.Len() < minReviewers && unusedLeaves.Len() > 0 {
        if r := selector(&unusedLeaves); r != "" {
            reviewers.Insert(1, r)
        } else {
            break
        }
    }

### [should-fix] err shadowing in handleStatus silently drops handle() errors
- resolved_in: `45d08fc6b`
- resolution: Both rifle and blunderbuss now use `if err := handle(...); err != nil` (block-scoped `:=` that doesn't shadow) and collect errors via `utilerrors.NewAggregate`. Also changed `return nil` to `continue` for draft/ignored/already-reviewed PRs, fixing the early-exit issue raised in open questions.

### [question] validateMutuallyExclusivePlugins with old-format configs
- resolved_in: `45d08fc6b`
- resolution: `validateMutuallyExclusivePlugins` now detects org-vs-repo conflicts (e.g. org has blunderbuss, org/repo has rifle) and respects `ExcludedRepos`. Six test cases added covering no-conflict, same-entry conflict, org/repo cross-conflict, excluded-repo, and separate-orgs scenarios.

## Checked

- `GetBlame` GraphQL query structure and nil-safe User handling in `client.go:4459-4491`
- `FallbackReviewersClient` correctly delegates `RequiredReviewers` (unlike old unexported version)
- `repoowners.RepoOwner` implements `AllOwners()` required by `reviewer.OwnersClient`
- `fakegithub.FakeClient.GetBlame` satisfies `CommitClient` interface
- `diverseFileSelection` round-robin logic and removed-file filtering in `blame.go`
- `parseDiffHunks` regex and range computation including pure-addition (count=0) case
- `intersectBlameWithChanges` overlap math and empty-login filtering
- Plugin registration for rifle (PullRequest, GenericComment, StatusEvent handlers)
- `BlameRange` type in `types.go`
- Test coverage for `selectBestReviewer`, `findFallbackReviewers`, `diverseFileSelection`, blame scoring
- handleStatus error handling fix: correct use of `utilerrors.NewAggregate`, `continue` instead of `return nil`
- `validateMutuallyExclusivePlugins` org/repo cross-check logic and ExcludedRepos handling
- `TestValidateMutuallyExclusivePlugins` covers all relevant scenarios
- `else { break }` fix in `GetReviewers` correctly terminates both selector loops
- New documentation for rifle in `approve/approvers/_index.md`: algorithm description, API quota note, mutual-exclusion note
- `GetMergeBase` REST call path (`/repos/{org}/{repo}/compare/{base}...{head}`), error handling, and empty-SHA guard in `client.go`
- Merge-base resolution and fallback-to-base-ref logic in `handle` (`rifle.go:276-286`)
- `TestHandleRifleBlamesMergeBase` covers success, error, and empty-merge-base fallback scenarios; `TestGetMergeBase` covers the REST client call

## Open questions

- Would it make more sense to add blame-based scoring as a mode within blunderbuss rather than a separate plugin? The event-handling code is identical; only the selector strategy differs. (Not addressed before merge; low priority now that the plugin has shipped.)

## Discussion (informational, no action needed)

- stevekuznetsov asked about additional GitHub API quota load (2026-07-22). Author (smg247) answered: up to 20 additional REST `GetBlame` calls per PR (capped by `maxBlameFiles`), `IsUserBusy` GraphQL calls unchanged from blunderbuss and deduplicated via `busyReviewers`. This is now also documented in the doc update. (Note: merge-base resolution adds one further REST call per PR event, on top of this estimate.)
- PR self-approved by author (kubernetes-prow bot APPROVALNOTIFIER, 2026-07-22T16:51:23Z).
- Prucek left an inline comment on `pkg/plugins/rifle/rifle.go:279` (2026-08-21T07:58:04Z) and a COMMENTED review (2026-08-21T09:14:19Z, no body) raising the base-ref/merge-base line-number concern, resolved in `76856c96e` (see Resolved).
- Prucek commented `/test pull-prow-integration` and `/lgtm` (2026-08-21T14:53:44Z) after the fix landed. PR merged at `76856c96e99c09c35cafce2e1998d6781db5da88`.

## Followups

Post-merge opportunities recorded by `/review:followup` on 2026-10-25 for PR #787 (merged to `main` as `a619408973827f5a8846fc61b4731156bde5fb0c`). Five accepted — four `deliver`, one `probe` — and one skipped (the `approvers/_index.md` fallback/API-quota wording, candidate 4).

### [should] deferred-review: make findFallbackReviewers respect use_status_availability
- where: `pkg/plugins/rifle/rifle.go:425-474`, called from `handle` at `:343-351`
- mode: deliver
- why: the unresolved `should-fix` finding above. With `use_status_availability: true`, `selectBestReviewer` skips busy users but `findFallbackReviewers` assigns them, so the setting silently stops working when OWNERS yields too few reviewers.

```text
In `kubernetes-sigs/prow`, following PR #787 ("rifle: smart reviewer selection; alternative to blunderbuss", merged to `main` as a619408973827f5a8846fc61b4731156bde5fb0c), make rifle's reviewer fallback honour `use_status_availability`.

Background: `pkg/plugins/rifle/rifle.go` has two selection paths. `selectBestReviewer` (around lines 369-414) skips candidates already in `busyReviewers` and, when `useStatusAvailability` is true, calls `reviewer.IsUserBusy`, caching the answer in `busyReviewers`. `findFallbackReviewers` (around lines 425-474), called from `handle` at around lines 343-351 when OWNERS-based selection yields fewer than `request_count` reviewers, performs no availability check at all: it ranks `oc.AllOwners()` by blame score, then fills the remainder uniformly at random.

Task:
1. Pass the GitHub client, the `useStatusAvailability` value, and the `*sets.Set[string]` busy cache into `findFallbackReviewers` (or restructure both paths behind one shared availability filter).
2. In both the blame-ranked loop and the uniform-random fill loop, skip users already in `busyReviewers`; when `useStatusAvailability` is true, call `reviewer.IsUserBusy` for each candidate considered, mirroring `selectBestReviewer` (log a warning on error and treat that user as available; insert busy users into `busyReviewers` so they are never queried twice).
3. Return fewer than `needed` reviewers rather than assigning a busy user — `handle` already tolerates a short list.
4. Keep the user-facing behaviour otherwise unchanged: blame-ranked candidates first, then random fill.

Acceptance criteria:
- With `use_status_availability: true` and the top-scored fallback candidates answering `IsUserBusy` as busy, `handle` requests reviews only from unbusy owners.
- New unit tests in `pkg/plugins/rifle/rifle_test.go` cover the all-candidates-busy case (no fallback reviewers) and the mixed case (busy skipped, unbusy assigned).
- No additional `IsUserBusy` call is issued for a user already present in `busyReviewers`.
- `go test ./pkg/plugins/rifle/...` and `go build ./pkg/plugins/rifle/...` pass.

Scope guard: do not change blunderbuss, the scoring weights, or the blame pipeline; do not add a config option; do not restructure the event handlers.
```

### [should] codegen: document the rifle config in plugin-config-documented.yaml
- where: `pkg/plugins/plugin-config-documented.yaml` (generated by `hack/gen-prow-documented/main.go`)
- mode: deliver
- why: `plugins.Rifle` was added without regenerating the config reference, so there is no top-level `rifle:` key. The same regeneration also adds the missing `max_request_count` to the `blunderbuss:` block. CI missed it because `hack/make-rules/verify/codegen.sh` copies `pkg/plugins` but never diffs it.

```text
In `kubernetes-sigs/prow`, following PR #787 ("rifle: smart reviewer selection; alternative to blunderbuss", merged to `main` as a619408973827f5a8846fc61b4731156bde5fb0c), regenerate the auto-generated plugin config reference so the new `rifle` section is documented.

Background: `pkg/plugins/plugin-config-documented.yaml` is generated by `go run ./hack/gen-prow-documented` (invoked from `hack/make-rules/update/codegen.sh`). PR #787 added the `plugins.Rifle` struct in `pkg/plugins/config.go` but did not regenerate, so the YAML has no top-level `rifle:` key. The same file is also missing `max_request_count` in its `blunderbuss:` block even though `plugins.Blunderbuss.MaxReviewerCount` exists.

Task:
1. From the repo root, run `go run ./hack/gen-prow-documented`.
2. Inspect the resulting diff of `pkg/plugins/plugin-config-documented.yaml`. Do not hand-edit the file.
3. Keep only genuinely regenerated content (`pkg/config/prow-config-documented.yaml` should come back unchanged).

Acceptance criteria:
- `pkg/plugins/plugin-config-documented.yaml` has a top-level `rifle:` block, alphabetically between `require_matching_label:` and `size:`, documenting the same fields as `blunderbuss:` (`exclude_approvers`, `ignore_authors`, `ignore_drafts`, `max_request_count`, `request_count`, `use_status_availability`, `wait_for_status`).
- The `blunderbuss:` block now also shows `max_request_count`.
- A second `go run ./hack/gen-prow-documented` produces no further diff.
- No Go source changes are made.

Scope guard: do not change the generator, the plugin structs, the docs page, or the set of config fields. Mention the `verify/codegen.sh` diff gap in the PR description only, not as a code change.
```

### [should] tech-debt: deduplicate the rifle/blunderbuss event plumbing and config
- where: `pkg/plugins/rifle/rifle.go:91-367`, `pkg/plugins/blunderbuss/blunderbuss.go:81-353`, `pkg/plugins/config.go:197-283`, `:1444-1456`, `:1699-1712`
- mode: deliver
- why: the other unresolved `should-fix`. Handler bodies, config structs, validation and `WaitForStatus` regexp compilation are duplicated verbatim, so every fix must be applied twice — candidate 1 being the latest example.

```text
In `kubernetes-sigs/prow`, following PR #787 ("rifle: smart reviewer selection; alternative to blunderbuss", merged to `main` as a619408973827f5a8846fc61b4731156bde5fb0c), deduplicate the code shared by the `blunderbuss` and `rifle` plugins.

Background: `pkg/plugins/rifle/rifle.go` (lines ~91-367) and `pkg/plugins/blunderbuss/blunderbuss.go` (lines ~81-353) contain near-identical `handlePullRequestEvent`/`handlePullRequest`, `handleGenericCommentEvent`/`handleGenericComment`, `handleStatusEvent`/`handleStatus`, and `handle` implementations. The only differences are the config type (`plugins.Rifle` vs `plugins.Blunderbuss`) and the selector strategy handed to `reviewer.GetReviewers`. In `pkg/plugins/config.go` the `Rifle` struct (~245-283) is field-for-field identical to `Blunderbuss` (~197-244), `validateRifle` (~1451) is identical to `validateBlunderbuss` (~1444), and `compileRegexpsAndDurations` compiles the same `WaitForStatus` description regexp twice (~1699-1712). The mutual-exclusion validator exists only because the plugin names stay separate.

Task:
1. Extract the shared event plumbing into a reusable place — prefer `pkg/reviewer` (which already hosts `GetReviewers`, `FindReviewer`, `FallbackReviewersClient`) or a new small package under `pkg/plugins/`. Parameterize it by the plugin config and by a selector/strategy: blunderbuss supplies `reviewer.FindReviewer`, rifle supplies its blame scorer with the random fallback. Rifle's `GetMergeBase`/`GetBlame` work stays in rifle.
2. Collapse the duplicated config shape into one shared struct (type alias or an embedded `ReviewerSelectionConfig`) while keeping every JSON key (`request_count`, `max_request_count`, `exclude_approvers`, `use_status_availability`, `ignore_drafts`, `ignore_authors`, `wait_for_status`) and the generated docs identical. Keep the `Configuration.Blunderbuss` and `Configuration.Rifle` field names.
3. Replace the duplicated validation and regexp-compilation blocks with one code path each.
4. Keep both plugin names, registration, help providers, and the mutual-exclusion validation.

Acceptance criteria:
- `rifle.go` and `blunderbuss.go` are reduced to registration, help, and strategy wiring — no duplicated event-handler bodies.
- `go test ./pkg/plugins/... ./pkg/reviewer/... ./cmd/checkconfig/...` passes, including blunderbuss's handler tests (`TestHandlePullRequest`, `TestHandleGenericComment`, `TestHandleStatus`, `TestHandle*Event`) and rifle's scoring tests.
- `go run ./hack/gen-prow-documented` yields no config-shape diff (JSON keys unchanged).
- No behaviour change: same events handled, same gating, same log messages where practical.

Scope guard: do not merge rifle into blunderbuss as a single plugin (separate product decision), do not change selection algorithms or scoring, do not add features. Keep the diff mechanical and reviewable.
```

### [could] tests: probe rifle's copied event-handler paths
- where: `pkg/plugins/rifle/rifle.go:91-257`, `pkg/plugins/rifle/rifle_test.go`
- mode: probe
- why: rifle tests the blame/scoring core but none of the ~150 duplicated handler lines (`/cc` skip, `wait_for_status` suppression, draft/ignore-authors, `/auto-cc`, status-event matching, non-head-SHA and already-requested-reviewer skips, multi-PR error aggregation) that blunderbuss covers.

```text
In `kubernetes-sigs/prow`, following PR #787 ("rifle: smart reviewer selection; alternative to blunderbuss", merged to `main` as a619408973827f5a8846fc61b4731156bde5fb0c), probe rifle's event-handler paths for bugs.

Context: `pkg/plugins/rifle/rifle.go` copies blunderbuss's handlers — `handlePullRequestEvent`/`handlePullRequest` (~91-124), `handleGenericCommentEvent`/`handleGenericComment` (~126-166), `handleStatusEvent`/`handleStatus` (~168-257), `handle` (~258-367) — but only the blame/scoring paths are tested (`pkg/plugins/rifle/rifle_test.go:416,511`). Blunderbuss covers all of them; use `pkg/plugins/blunderbuss/blunderbuss_test.go` (`TestHandlePullRequest`, `TestHandleGenericComment`, `TestHandleStatus`, `TestHandleGenericCommentEvent`, `TestHandlePullRequestEvent`, `TestHandleStatusEvent`) as the reference for fake setup.

This is a probe. Write throwaway Go tests (in `pkg/plugins/rifle/` or a scratch package, using `fakegithub` and the existing rifle fake owners client) covering:
- `handlePullRequest`: opened vs. `synchronize`; `/cc` in the PR body suppresses; `WaitForStatus != nil` suppresses immediate handling; draft + `ignore_drafts`; `ignore_authors`.
- `handleGenericComment`: created vs. edited; non-PR; closed PR; `/auto-cc` present vs. absent.
- `handleStatus`: matching vs. non-matching context, state, and description regexp; PR found for the SHA vs. `pr.Head.SHA` mismatch; `RequestedReviewers` already populated; draft and `ignore_authors`; two matching PRs where one `GetPullRequest` errors (error aggregated, the other still handled).
- `handle` end-to-end with every reviewer candidate busy, proving the selection loops terminate and request no reviews (regression net for the infinite-loop bug fixed before merge).

Run them with `go test ./pkg/plugins/rifle/... -count=1`. For each failure decide whether it is a real implementation bug or a wrong expectation/test mistake, and show the evidence from the code path. Apply only the minimal fix for a confirmed real bug.

Acceptance criteria:
- Every behaviour above is exercised; every failure is classified with evidence.
- If a real bug is found, the minimal fix is applied and nothing else.
- If no real bug is found, no files are changed.
- Probe tests are deleted before finishing.

Scope guard: do not submit the probe tests, do not refactor, do not fix test-only or style problems, do not touch blunderbuss. Report each failing case (input, expected, actual) so it can be kept as a regression test if wanted.
```

### [could] nit: replace the hand-rolled insertion sort in selectBestReviewer
- where: `pkg/plugins/rifle/rifle.go:391-398`
- mode: deliver
- why: the last open review finding; a four-line reimplementation of `sort.SliceStable` with a descending comparator, in a file that already imports `sort`.

```text
In `kubernetes-sigs/prow`, following PR #787 ("rifle: smart reviewer selection; alternative to blunderbuss", merged to `main` as a619408973827f5a8846fc61b4731156bde5fb0c), simplify ranking in `selectBestReviewer`.

Background: in `pkg/plugins/rifle/rifle.go` around lines 391-398, candidates are shuffled with `rand.Shuffle` and then ordered with a hand-written insertion sort:

    for i := 1; i < len(ranked); i++ {
        for j := i; j > 0 && ranked[j].score > ranked[j-1].score; j-- {
            ranked[j], ranked[j-1] = ranked[j-1], ranked[j]
        }
    }

Task: replace that loop with `sort.SliceStable(ranked, func(i, j int) bool { return ranked[i].score > ranked[j].score })`, keeping the `rand.Shuffle` before it so equal scores stay randomly ordered.

Acceptance criteria:
- `TestSelectBestReviewer`, `TestSelectBestReviewerSkipsBusy`, and `TestSelectBestReviewerNoScores` pass unchanged with `go test ./pkg/plugins/rifle/...`.
- `go vet ./pkg/plugins/rifle/...` and gofmt are clean.

Scope guard: change only that sorting loop; do not touch `findFallbackReviewers`, scoring, or the handler code.
```
