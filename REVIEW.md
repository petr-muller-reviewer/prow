---
pr: kubernetes-sigs/prow#909
title: "blunderbuss, rifle: count existing eligible reviewers toward the reviewer count"
head_sha: bf10026abb714fae5dfcc1f9353046bdc6d5980f
base: main
reviewed_at: 2026-08-31T22:28:26Z
verdict: needs-discussion
---

## Summary

Reviewed the self-contained commit `bf10026ab` (`git diff HEAD~1..HEAD`, 845 lines) touching
`pkg/plugins/blunderbuss/blunderbuss.go`, `pkg/plugins/rifle/rifle.go`, and
`pkg/reviewer/reviewer.go`. The PR makes blunderbuss/rifle count pre-existing eligible
reviewers/approvers toward `reviewer_count`/`max_reviewer_count`, instead of always adding
`reviewer_count` new reviewers on top. Quota/exclusion arithmetic is sound and covered by new
tests (case normalization, approver-only eligibility, `maxReviewerCount` interaction,
`reviewerCount: 0` no-op). Main concern is duplicated orchestration between the two plugins
and an inconsistency in required-reviewer collection introduced by the new "quota already
met" branch.

## Findings

### [should-fix] Required-reviewer collection diverges from GetReviewers's dedup
- where: `pkg/plugins/blunderbuss/blunderbuss.go:297`, `pkg/plugins/rifle/rifle.go:361`
- concern: The new "quota already met by existing reviewers" branch collects required
  reviewers by unioning `oc.RequiredReviewers(change.Filename)` over every changed file,
  while `reviewer.GetReviewers`'s existing loop (`pkg/reviewer/reviewer.go:102-107`) dedups
  files by their nearest reviewers-OWNERS directory (`ownersSeen`) before collecting required
  reviewers. `RequiredReviewers` walks a separate, independent directory chain per file, so
  two files sharing a nearest reviewers-OWNERS dir can still have distinct required-reviewer
  entries in their own subdirectory OWNERS files.
- failure_scenario: If the quota still needs filling, `GetReviewers`'s `ownersSeen` dedup
  silently drops the second file's required reviewer. If pre-existing /cc'd reviewers already
  satisfy the quota, the new manual branch (no dedup) picks up both files' required
  reviewers. The same PR/file set ends up requesting a different set of required reviewers
  purely depending on whether existing reviewers happened to fill the quota.

### [should-fix] Quota-orchestration logic duplicated between blunderbuss and rifle
- where: `pkg/plugins/blunderbuss/blunderbuss.go:290-320` (approx.), `pkg/plugins/rifle/rifle.go:340-370` (approx.)
- concern: The entire `existingEligible`/`neededReviewers`/`maxNewReviewers` orchestration,
  plus the required-reviewers-only block above, is duplicated near-identically between the
  two plugins instead of living once in `pkg/reviewer`. Rifle's copy already needed one extra
  line blunderbuss's doesn't (`excludeFromFallback.Union(existingEligible)` for its
  blame-fallback stage), showing the copies can already drift.
- failure_scenario: A future fix to quota accounting (including the dedup inconsistency
  above) must be applied identically in three places (`GetReviewers` plus two plugin copies)
  or the two plugins silently diverge — the same class of bug this PR set out to fix.

### [question] max_reviewer_count still not a hard ceiling
- where: `pkg/plugins/blunderbuss/blunderbuss.go:308`, `pkg/plugins/rifle/rifle.go` (same computation)
- concern: `maxNewReviewers` only discounts `existingEligible` (pre-existing reviewers that
  are also in the OWNERS reviewer/approver pool), not all currently-requested reviewers, so
  `max_reviewer_count` is still not a hard cap on the PR's total requested-reviewer count when
  reviewers already requested aren't in the OWNERS pool.
- failure_scenario: A PR already has 2 manually /cc'd reviewers who are not in the OWNERS pool
  for the changed files, with `max_reviewer_count=2`. `existingEligible.Len()==0`, so
  `maxNewReviewers` stays 2 and blunderbuss adds 2 more, leaving 4 total requested reviewers
  despite the configured cap. Not a regression (old code had the identical limitation), but
  this PR revisits this exact accounting without closing the gap — worth asking the author
  whether that's intentional/out-of-scope.

### [question] GetReviewers short-circuits before collecting required reviewers at minReviewers==0
- where: `pkg/reviewer/reviewer.go:98`
- concern: `GetReviewers` returns early when `minReviewers==0` — exactly the case the new
  "quota already satisfied" scenario hits — forcing both plugins to bolt on a second,
  independent required-reviewer computation rather than `GetReviewers` handling
  `minReviewers==0` by skipping only reviewer selection while still collecting required
  reviewers.
- failure_scenario: This is the root cause of the two `should-fix` findings above. Fixing
  `GetReviewers` to always collect required reviewers consistently (regardless of
  `minReviewers`) would let both plugins reuse it instead of hand-rolling a duplicate branch.

### [nit] rifle_test.go missing a required-reviewers-at-quota-met case
- where: `pkg/plugins/rifle/rifle_test.go:475`
- concern: `TestHandleRifleWithExistingRequestedReviewers` has no case for "quota already met
  by existing reviewers AND a `RequiredReviewers` entry exists," unlike
  `blunderbuss_test.go`'s equivalent "quota satisfied but required reviewers still requested"
  case.
- failure_scenario: A regression in rifle.go's else-if branch (`rifle.go:361-369`) — e.g.
  wrong client used for `RequiredReviewers`, or a copy-paste slip — would go undetected by
  rifle's test suite even though the identical branch in blunderbuss is covered.

### [nit] Repeated OWNERS walks across overlapping loops
- where: `pkg/reviewer/reviewer.go:156` (`EligibleRequestedReviewers`), plus rifle's own `allReviewerCandidates`/`allApproverCandidates` loop
- concern: `EligibleRequestedReviewers` iterates changed files calling
  `oc.Reviewers()`/`oc.Approvers()` to build the eligibility pool, duplicating OWNERS-lookup
  work already done by `GetReviewers`'s own per-file loop and by rifle's separate candidate
  loop right below it.
- failure_scenario: For PRs touching many files, three-to-four overlapping passes over the
  same file list and OWNERS data occur per plugin invocation — wasted OWNERS-walk work that
  scales with file count on every PR event, though not a correctness issue.

## Checked

- Quota/exclusion arithmetic itself (subtracting existing eligible reviewers from the target
  count) is correct and covered by new tests: case normalization, approver-only eligibility,
  `maxReviewerCount` interaction, `reviewerCount: 0` no-op.
- Used the single-commit diff (`git diff HEAD~1..HEAD`), not `main...HEAD` — local `main` was
  far stale relative to this branch.

## Open questions

- Is the `max_reviewer_count` gap (finding above) intentional/out of scope for this PR, or
  should it be tightened here since the accounting is already being revisited?
- Would you be open to moving the `existingEligible`/`neededReviewers`/required-reviewers
  orchestration into `pkg/reviewer.GetReviewers` itself, to avoid the blunderbuss/rifle
  duplication (and the dedup inconsistency it introduces)?
