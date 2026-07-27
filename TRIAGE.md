---
issue: kubernetes-sigs/prow#541
title: "Tide should force retest a suspiciously passing required job on mergeable PRs"
state: closed
labels: kind/feature, lifecycle/rotten, area/plugins, area/tide
main_sha: e601a1ffafd7d8d3a781238a4c5f4233d6248f68
triaged_at: 2026-07-27T00:32:25Z
verdict: needs-discussion
---

## Findings

### [cause] Prior triage's root cause claim is contradicted by existing code
- detail: A prior full triage of this same issue (2026-02-04, `origin/issue-triage-541:ISSUE-TRIAGE.md`, posted as an issue comment) concluded `accumulate()` trusts already-passing required contexts without verifying they're backed by a real ProwJob or legitimate `/override`. Re-checking against current `main`, `prowJobsFromContexts()` only synthesizes a ProwJob for a passing context if its description carries a matching baseSHA (`config.BaseSHAFromContextDescription`) or an explicit skip-retest sentinel (`config.IsSkipRetest`). Anything else is excluded from `psStates` in `accumulate()` and falls into `missingTests`, forcing a retest.
- evidence: `pkg/tide/tide.go:1044-1076` (`prowJobsFromContexts`), `pkg/tide/tide.go:1080-1138` (`accumulate`, esp. 1112-1115).

### [cause] The verification gate predates the issue by over 3 years
- detail: Blamed the `baseSHAForContext != "" && baseSHAForContext == baseSHA` condition in `prowJobsFromContexts` back through `c8a0d6bc3` → `5a3782943` to `a1e8b25f4` ("Tide Controller interacts with GitHub by interfaces", 2022-06-02). The issue was filed 2025-10-31, the #540 incident occurred 2025-10-14. The protective mechanism #541 asks for already existed when both were filed.
- evidence: `git log -L1044,1076:pkg/tide/tide.go`; commit `a1e8b25f4c1067ac9a7ce9a9ef134ea16b28f1e0` (2022-06-02).

### [reproducibility] #540's actual repro would already be caught by the existing gate
- detail: #540's `status-reconciler` set contexts with description `"Context retired without replacement."` — no baseSHA encoding. `BaseSHAFromContextDescription` returns `""` for that string, so the context fails the verification gate, is excluded from `psStates`, and `accumulate()` puts it in `missingTests`, forcing a retest rather than merging on the falsely-retired status. This has not been confirmed by executing a test — only by reading the code and matching it against the logged reproduction in #540.
- evidence: issue #540 body (log excerpt: `CreateStatus(... {success Context retired without replacement. ci/prow/odh-dashboard-pr-image-mirror})`); `pkg/tide/tide.go:1050-1059`.

### [related-code] Merge decisions are gated on the verified path, not the raw one
- where: `pkg/tide/tide.go:845-889` (`isPassingTests`/`unsuccessfulContexts`), `pkg/tide/tide.go:1545,1561-1562` (`takeAction`), `pkg/tide/tide.go:1783` (`accumulate` call site)
- excerpt: |
    // isPassingTests returns whether or not all contexts set on the PR except for
    // the tide pool context are passing.
    func (c *syncController) isPassingTests(log *logrus.Entry, pr *CodeReviewCommon, cc contextChecker) bool {
- detail: `isPassingTests`/`filterPR` use raw, unverified GitHub status but only decide pool membership. `takeAction` only merges PRs from the baseSHA-verified `successes` bucket produced by `accumulate()`. The unverified check never gates an actual merge decision.

### [related-pr] PR #778 incidentally shipped the one remaining gap (override baseSHA encoding) after the issue was already closed
- ref: kubernetes-sigs/prow#778
- relevance: The prior triage's "Part 1" recommendation — make `/override`-created statuses encode baseSHA like normal Prow results — landed via commit `c8a0d6bc3` ("Move skip-retest contract to config and embed baseSHA in override descriptions", 2026-06-30) as part of unrelated work on `/override-sticky` (PR #778, merged 2026-07-14). Confirmed current `pkg/plugins/override/override.go:564`: `status.Description = config.ContextDescriptionWithBaseSha(descFn(user), baseSHA)`. This merged 10 days *after* #541 was auto-closed (2026-07-04) — nobody connected the two.

### [related-issue] #540 is the motivating incident
- ref: kubernetes-sigs/prow#540
- relevance: Open issue describing the haywire `status-reconciler` that falsely retired job contexts to "passing"; #541 is the follow-up asking Tide to protect against exactly this kind of false-positive status.

## Checked
- Confirmed `override.go:564` currently encodes baseSHA via `config.ContextDescriptionWithBaseSha` (prior triage's Part 1 fix already shipped).
- Blamed `prowJobsFromContexts`'s baseSHA-matching condition back through `c8a0d6bc3` → `5a3782943` → `a1e8b25f4` (2022) to establish it predates both the issue and #778.
- Read `filterPR`/`isPassingTests`/`unsuccessfulContexts` (`pkg/tide/tide.go:755-793,845-889`) to confirm the unverified path only affects pool membership, not merge eligibility.
- Matched #540's logged status description (`"Context retired without replacement."`) against the current baseSHA-matching logic by inspection (not by running a test).
- Located a prior, differently-formatted full triage on `origin/issue-triage-541:ISSUE-TRIAGE.md` (2026-02-04) and used it as a research starting point rather than taking its conclusions at face value.
- Confirmed via `gh issue view` that #541 was closed 2026-07-04 by `k8s-triage-robot` as `not-planned`, purely on the stale/rotten/close inactivity timer — no maintainer judged it invalid.

## Next steps
- Reopen #541 (`/reopen`) — it was closed by the staleness bot on inactivity, not on the merits.
- Write a regression test in `pkg/tide/tide_test.go` reproducing #540's exact context description (no baseSHA) against `accumulate()`/`prowJobsFromContexts()`, to confirm by execution that it lands in `missingTests` and forces a retest.
- If confirmed: comment on #541 citing `tide.go:1044-1076`, the `a1e8b25f4` history, and PR #778's incidental baseSHA fix in `override.go`, then close as already-addressed (landing the new regression test in the closing PR).
- If the test reveals a gap instead: narrow down which code path lets an unverified context through to `takeAction`, and re-scope a fix accordingly.
- Drop `lifecycle/rotten`; apply `/priority backlog` (not `important-soon` — this is not an active vulnerability per findings above).

## Open questions
- Is there a scenario (e.g. non-required contexts becoming required mid-flight, `Tide.RequiredStatuses`-style config) where `accumulate()`'s ProwJob-matching is bypassed and raw GitHub state gates the merge decision? Not identified in this pass, not exhaustively ruled out.
- Did the original author intend "suspiciously passing" to cover more than the baseSHA-mismatch case already handled — e.g. a context with a *stale but matching* baseSHA that's otherwise wrong?
