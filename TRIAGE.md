---
issue: kubernetes-sigs/prow#400
title: "`tide` merge queue stalls when unresolved comments exist"
state: open
labels: kind/bug, lifecycle/stale, area/tide
main_sha: 7f60ede4f88faf9dbbb96ed6cd5e73390afc53be
triaged_at: 2026-10-28T01:02:25Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 3
recommended_labels: [kind/bug, area/tide]
advice:
  advised_at: 2026-10-28T01:08:49Z
  based_on_triaged_at: 2026-10-28T01:02:25Z
---

## Initial validation

**Assessment**: LEGITIMATE

### Analysis
The reporter aevyrie describes a concrete Tide failure: require resolved review conversations in GitHub, admit a PR with unresolved conversations to the merge pool, and observe failed merges preventing other candidates from progressing. Expected behavior is to skip the blocked candidate. This is a scheduler robustness issue in this repository, even though branch protection triggers it.

**Issue Category**: Bug

**Repository Scope Check**:
- Component: Tide; maintained here in `pkg/tide/github.go`, `pkg/tide/tide.go`, and `pkg/tide/status.go`.

**Information Completeness**:
- Enough information for code analysis and a regression scenario.
- Missing: affected deployment version, effective merge-block policy, and a fresh post-#579 reproduction/log. No live reproduction has been performed.

### Recommendation
Keep open while checking current behavior. #269 is an older related report about requested changes; petr-muller linked it on 2025-08-23 and aevyrie agreed the root cause overlaps on 2025-09-02. Treat #400 as the unresolved-conversation regression case rather than assuming its particular branch rule is verified by review-count coverage. #673 reports the same scheduling failure with insufficient approvals; it is closed, but linked PR #674 is still open, so closure does not establish a shipped fix.

**Suggested Action**: Keep open and continue triage; assess existing fixes before soliciting another implementation.

## Research findings

### How it works today
`PullRequest` queries `mergeStateStatus` (`pkg/tide/tide.go:2061-2083`). Sync filters pools with `isAllowedToMerge` (`pkg/tide/tide.go:647`); the checker rejects BLOCKED only for policy `block` (`pkg/tide/github.go:742-785`). Policy resolves repository, organization, then global overrides and defaults to `permit` (`pkg/config/tide.go:513-527`). Under permit the PR status can say "In merge pool (despite BLOCKED)" (`pkg/tide/status.go:369-385`).

`accumulate` classifies candidates from presubmit results without previous merge outcomes (`pkg/tide/tide.go:1150-1210`). `pickHighestPriorityPR` chooses the lowest number in the first matching priority tier (`pkg/tide/tide.go:982-1008`), and `takeAction` attempts one successful candidate per cycle (`pkg/tide/tide.go:1615-1644`).

### Root cause
With permit/ignore, a PR can satisfy Tide while GitHub rejects its merge for unresolved conversations. `tryMerge` returns a wrapped UnmergablePRError and permits proceeding within an existing batch (`pkg/tide/tide.go:1497-1555`); it does not record state for subsequent cycles. The next accumulation/selection can pick the same candidate indefinitely, starving other single-merge candidates in that org/repo/branch subpool. This does not imply every PR in every pool or an already-running batch is blocked.

Confidence is high in the control-flow diagnosis, conditional on GitHub returning UnmergablePRError for the reporter's branch rule. No live GitHub reproduction was performed; version, effective policy, and a fresh two-candidate reproduction remain unverified. `block` offers an existing opt-in mitigation when GitHub reports BLOCKED, but cannot safely become unconditional: petr-muller's 2026-03-16 comment on #269 explains #537 was reverted because GitHub Apps with bypass rights can merge legitimately despite BLOCKED.

Current visibility is better than the old triage stated: merge errors are user-facing in Tide history (`pkg/tide/tide.go:1866-1908`), and permit status warns about BLOCKED. Neither warning prevents repeat selection.

### Tests and docs
Coverage is partial. `TestIsAllowedToMerge_ReviewDecision` covers block/permit/ignore/default behavior (`pkg/tide/tide_test.go:1560-1655`); `TestGitHubMergeBlocksPolicy` exercises status policy (`pkg/tide/status_test.go:1636-1700`). The existing batch-unmergeable case (`pkg/tide/tide_test.go:2113-2122`) verifies continuation within one batch, not progress across syncs. Config documentation describes the policy in `pkg/config/prow-config-documented.yaml`.

PR #674 head `d6d08d0b2b40c9ef3fee39d3886a44be0c993f23` now uses a counter initialized to three and aged at each Sync, not its body's outdated five-minute TTL. Its diff includes two-candidate progress, wrapped-error detection, aging/new-SHA cleanup, failed-batch recording, and new-batch filtering tests. Tests were inspected, not executed. Counting down before selection appears to skip two subsequent syncs and permit the third; review that boundary against the stated intent.

### Proposed approach
**Recommended**: Review and converge #674's bounded exclusion after actual GitHub rejection. Keep the existing merge-block policy semantics and allow other candidates to progress after an UnmergablePRError; scope state by repository, PR and head SHA, expire it, and cover batch paths as well as direct merges. This learns from actual API rejection and preserves successful App bypass behavior.
- Trade-offs: resolving a conversation without a new commit waits for exclusion expiry; state is in memory and resets on restart. Confirm the cycle-count boundary and useful PR-facing diagnostics.
- Backwards compatibility: preserves permit/ignore and App bypass eligibility; changes scheduling after rejected merges. Shared controller code also serves Gerrit, so preserve provider isolation.
- Testing needed: unresolved-conversation two-candidate reproduction; expiry after blocker clears without a push; new SHA; wrapped errors; batch and direct merge progress; normal App-bypass success. Existing PR tests cover parts of this matrix but do not prove live GitHub semantics.

**Alternative**: For deployments that intentionally enforce every GitHub blocking rule, set repository-scoped `tide.github_merge_blocks_policy` to `block` and verify status/merging after resolving a conversation. This can avoid the failed attempt, but is unsuitable as a global default for bypass-dependent deployments and leaves unexpected rejection handling unaddressed.


## Effort assessment

**Effort Level**: 3 - Large

The implementation can remain within Tide, but it changes core merge scheduling, shared state across concurrent subpools, and deployment-visible behavior. GitHub App bypass compatibility raises the required expertise even with #674 already in flight.

- **Solution certainty**: Some open questions — bounded exclusion is a clear direction; confirm cycle boundary, batch behavior, recovery latency and diagnostics.
- **Blast radius**: Core paths and deployment-visible scheduling; preserve successful bypass merges and other providers.
- **Expertise**: Tide single/batch merge flow, wrapped Go errors, controller concurrency, and GitHub rule/bypass semantics.
- **Scope estimate**: Medium — controller state and candidate selection plus regression tests for rejection, recovery, SHA changes, batches and bypass success.

**Recommended labels**: `kind/bug` (starvation after rejection), `area/tide` (scheduler). Neither good-first-issue nor help-wanted at this level; #674 already provides an implementation to review. No priority label without deployment-impact evidence.

**Guidance for contributors**: Start with #674 and #269's #537 revert discussion. Agree on retry semantics with Tide maintainers before creating a competing patch; ensure exclusions expire and cannot affect unrelated PRs or providers.

## Briefing summary

Briefed the maintainer on the legitimate Tide starvation report, current configurable BLOCKED-state handling, and the remaining lack of cross-sync merge-failure exclusions. Recommended reviewing #674's bounded, head-SHA-scoped exclusions while preserving GitHub App bypass behavior. Discussed partial existing coverage, the current patch's sync-count boundary and stale five-minute description, and Level 3 effort because this affects core scheduling and concurrent controller state.

Briefed maintainer on: 2026-10-28T01:02:25Z

Key questions asked:
- No substantive questions; the maintainer advanced through all seven slides.

Maintainer decision:
- No implementation, closure, label, or publication decision expressed. Recommendation is to review #674 against the unresolved-conversation case and obtain current deployment evidence.

## Findings

### [related-issue] Requested changes and insufficient approvals
- ref: kubernetes-sigs/prow#269 (open), kubernetes-sigs/prow#673 (closed)
- relevance: Same retry/scheduling mechanism under different GitHub rules; #673's linked PR #674 remains open.

### [related-pr] Configurable GitHub blocking enforcement
- ref: kubernetes-sigs/prow#579 (merged 2026-05-31)
- relevance: Adds mergeStateStatus and configurable ignore/permit/block policy; default permit preserves bypass compatibility.

### [related-pr] Retry cooldown in flight
- ref: kubernetes-sigs/prow#674 (open)
- relevance: Current diff proposes a head-SHA-keyed exclusion counted in sync cycles following UnmergablePRError; the body still describes five minutes.

### [cause] Repeat selection after rejected merges
- detail: Default permit admits BLOCKED candidates; no previous-rejection state changes accumulation or single-PR selection across cycles.
- evidence: `pkg/tide/github.go:742-785`; `pkg/tide/tide.go:982-1008,1150-1210,1615-1644`.

### [related-code] Existing policy and history visibility
- where: `pkg/config/tide.go:513-527`; `pkg/tide/status.go:369-385`; `pkg/tide/tide.go:1866-1908`
- excerpt: |
    return GitHubMergeBlocksPermit
    errorString = mf.historyMessage()

### [reproducibility] Code-supported, live reproduction outstanding
- detail: Require resolved conversations, use permit/ignore, make the blocked PR win priority among two passing candidates, and observe repeat rejection and starvation. Not executed against a live deployment.
- evidence: Reporter body and deterministic candidate selection; deployment version/policy missing.

## Checked
- Read current checker, accumulation, priority selection, merge/error handling and status/history paths; the old missing-GraphQL-field finding is obsolete.
- Inspected #674's current diff, including sync aging, wrapped errors and batch exclusions; its description is stale about the duration.
- Read #269's bypass/revert discussion; unconditional BLOCKED rejection is incompatible with some deployments.
- Full #400 body and paginated comments/timeline; latest comment is k8s-triage-robot's stale notice on 2026-10-21. No new substantive reporter confirmation.
- Similar-issue searches: `tide unresolved`, `tide stalls`; examined #269 and #673 bodies and #269's discussion.
- #400 cross-reference: gke-labs/kube-agents#1363 (closed downstream migration epic); does not establish upstream resolution.

## Next steps
- Review #674 head d6d08d0b2b40c9ef3fee39d3886a44be0c993f23 for #400's unresolved-conversation case; reconcile its three-count sync aging with the stale five-minute description and verify batch recovery.
- Ask for the affected Tide version, effective policy, merge rejection/history and a two-candidate reproduction; verify behavior after resolving a conversation without pushing.
- Consider repository-scoped block policy as an interim mitigation only after checking bypass requirements and Tide-context behavior.
- Keep kind/bug and area/tide; recommend manually removing lifecycle/stale after confirming continued ownership. Link #674 to #400 when publishing is explicitly requested.

## Advice

No issue activity since triage: #400 remains open with kind/bug, area/tide and lifecycle/stale; no new comments or cross-references, assignee or milestone. PR #674 remains open at d6d08d0b2b40c9ef3fee39d3886a44be0c993f23, currently MERGEABLE with no review decision. The saved next steps still apply. These are suggested actions; none has been executed.

1. **Link #674 and request current deployment evidence in one comment.** The timeline does not link the upstream fix, and the code-supported diagnosis still needs the reporter's version, policy and unresolved-conversation reproduction.

   ```bash
   gh issue comment 400 --repo kubernetes-sigs/prow --body-file - <<'EOF'
   > *This was generated by AI during triage.*

   PR #674 targets the repeated merge-rejection loop and is still open. Its current patch temporarily excludes rejected PRs by head SHA using sync-cycle counts, although its description still mentions five minutes. Please review it against this unresolved-conversation case, including recovery after resolving a conversation without pushing.

   @aevyrie, can you share the affected Tide version, effective github_merge_blocks_policy, whether the GitHub App relies on bypass rights, and merge-rejection logs or Tide history showing a blocked PR repeatedly selected while another PR in the same subpool is ready? If possible, confirm what happens after resolving the conversation without a new commit. #579 added opt-in block policy; the default permit policy still allows BLOCKED candidates.
   EOF
   ```

2. **Remove lifecycle/stale.** The fresh analysis identifies an actionable scheduling gap with a fix in flight; kind/bug and area/tide are already present. Level 3 effort does not warrant good first issue or help wanted. No milestone or priority target is established.

   ```bash
   gh issue edit 400 --repo kubernetes-sigs/prow --remove-label lifecycle/stale
   ```

3. **Review #674 before commissioning another implementation.** Check unresolved-conversation rejection, recovery without a push, the exclusion counter boundary, batch progress, and successful App bypass. MERGEABLE is not evidence of correctness or approval.

   ```bash
   gh pr diff 674 --repo kubernetes-sigs/prow
   ```

## Open questions
- What version and effective github_merge_blocks_policy does the affected deployment run, and does it rely on App bypass?
- Does unresolved-conversation rejection map to UnmergablePRError in that deployment, and do other candidates progress under block policy?
- What retry delay is acceptable after resolving a conversation without a push? Is #674 intended to skip two or three following syncs?
- Should exclusions appear in the PR status as well as logs/history?
