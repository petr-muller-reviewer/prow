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
