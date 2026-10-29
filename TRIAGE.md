---
issue: kubernetes-sigs/prow#474
title: "2 pr's are not compatible in merge pool"
state: open
labels: kind/bug, area/tide
main_sha: b99244a63a72e55978010020b225d27637bf4051
triaged_at: 2026-10-28T01:17:50Z
verdict: accepted
legitimacy: LEGITIMATE
effort: 3
recommended_labels: [kind/bug, area/tide]
advice:
  advised_at: 2026-10-28T01:19:37Z
  based_on_triaged_at: 2026-10-28T01:17:50Z
---

# Triage

## Verdict

**Accepted — keep open; effort level 3.**

The report concerns Tide's scheduling code and remains worth investigating. Current code permits a conditional repeated-batch loop, but the production cause is unverified: individual retesting can run while a batch is pending. PR #538 closed unmerged, and the newer overlapping #1041 was withdrawn. Capture a multi-sync reproduction before choosing a scheduling change; retain the existing `kind/bug` and `area/tide` labels.

## What the issue reports

- Two otherwise eligible PRs can be semantically incompatible when tested together.

- Tide reportedly repeats their batch without progressing to an individual merge.

- Maintainers expect one PR to be retested and merged, followed by retesting the other against the updated base.

- Individual results becoming stale after another merge is a suspected precondition; the precise incident trigger is unresolved.

- The issue reopened on October 28, 2026 after automatic closure; the proposed fix has not merged.

## Initial validation

**Assessment**: LEGITIMATE

### Analysis

[Issue #474](https://github.com/kubernetes-sigs/prow/issues/474) describes incompatible PRs repeatedly batch-tested without making progress. Maintainers expect Tide to retest and merge one PR individually. Discussion identifies stale individual results after the base advances as a relevant condition; the March 10 clarification explicitly leaves open why serial retesting does not rescue the pool. This is a valid Tide scheduling report, although its precise production trigger remains unproven.

The issue was automatically closed on October 17, 2026 and reopened by petr-muller on October 28 with a statement that the problem remains. This fresh triage replaces the stale July analysis, including its incorrect claim that PR #538 remains open.

**Issue Category**: Bug

**Repository Scope Check**:

- Component mentioned: Tide merge-pool scheduling.

- Exists in this repo: Yes, `pkg/tide/tide.go`.

- Relevant paths: `accumulateBatch`, `accumulate`, `pickBatch`, `takeAction`, and `syncSubpool` in `pkg/tide/tide.go`.

**Information Completeness**:

- Sufficient detail provided: Yes for investigation; not yet a verified reproduction against this checkout.

- Missing information: incident-time individual job states, pending batch states across syncs, deployment revision/configuration, and whether a required batch failure precedes any successful serial retest opportunity.

### Recommendation

Keep open. The closed, unmerged [PR #538](https://github.com/kubernetes-sigs/prow/pull/538) is prior work, not a shipped fix. [Issue #1041](https://github.com/kubernetes-sigs/prow/issues/1041) describes the same proposed loop in more detail but was closed by its author because its AI analysis was considered mistaken. Treat it as an unverified overlapping report, not independent confirmation or grounds to close this older record.

**Suggested Action**: Keep open and continue triage; gather a deterministic reproduction before selecting a fix.

## Research findings

### How it works today

`syncSubpool` accumulates individual and batch results, then chooses one action (`pkg/tide/tide.go:1859-1880`). Individual successes must have a matching current base (or skip-retest sentinel) when reconstructed from contexts (`pkg/tide/tide.go:1116-1129`); missing/failed required current-base jobs put a PR in `missings` (`pkg/tide/tide.go:1151-1210`).

Batch candidates need acceptable head contexts, but may have stale-base successes: `pickBatch` uses `isRetestEligible`, not current-base success classification (`pkg/tide/tide.go:1271-1327,1335-1381`). This is intentional retesting eligibility, not by itself a bug. `pickNewBatch` checks Git merge compatibility, which cannot detect semantic/test incompatibility (`pkg/tide/tide.go:1227-1263`).

`accumulateBatch` groups jobs by refs and combines the best results per context. A missing or failing required context makes the batch invalid; only successful and pending sets are returned (`pkg/tide/tide.go:1016-1113`). Importantly, missing jobs also use `failureState`, so adding a failure return naively would conflate incomplete job sets with actual CI failures. `takeAction` prefers a successful batch, then a ready individual PR when no batch is pending, then a new batch, then a serial retest (`pkg/tide/tide.go:1615-1665`).

`trigger` deduplicates batch jobs using an index keyed by job, repo, base, and PR numbers/SHAs. Completed unsuccessful jobs are excluded from that index and can be retried (`pkg/tide/tide.go:1557-1612,2259-2287`). A `TRIGGER_BATCH` history action alone does not prove every batch job was recreated: nonfailed jobs are reused.

### Root cause

**High confidence in the conditional control-flow mechanism; moderate confidence that it explains the reported incidents.** At a stable base, if all individual results need retesting, at least two PRs remain batch-eligible and merge-compatible, and no batch is classified pending or successful when each sync observes it, Tide can repeatedly select the same batch before reaching serial retesting. Fast failure of a required batch context before the following sync is one sufficient scenario. Because any required failure dominates the aggregate, other batch jobs can still be running while `batchPending` is empty.

**Correction to the previous triage:** ignoring failed batches alone does not establish a livelock. If a valid pending batch is observed and there are no serial successes or pendings, `takeAction` can start a serial retest; if a serial success exists and no batch is pending, Tide can merge it. These branches are explicitly tested. In PR #538's incident log, multiple pending-batch snapshots show WAIT, but the excerpt does not include individual buckets, so it does not establish why the serial branch was unavailable.

No runtime reproduction was executed, and no incident deployment revision or complete job/status timeline was supplied. The renewed statement that the bug remains is evidence to continue investigation, not proof of a particular root cause. The newer #1041 analysis was withdrawn and supplies no independent validation. History search for the action-order/failure-state constructs leads back to the 2024 source move, so no recent regression commit is established.

### Tests and docs

**Coverage: Partial.** `TestAccumulateBatch` covers failure, missing-required-job and success-after-failure behavior (`pkg/tide/tide_test.go:91-219`). `TestTakeActionV2` intentionally expects failed batch jobs to retry (`pkg/tide/tide_test.go:2053-2091`) and verifies serial retesting while a batch is pending (`pkg/tide/tide_test.go:2093-2113,2162-2186`). The examined coverage checks isolated decisions; it does not establish eventual progress across repeated failing-batch syncs. No tests were run for this analysis-only task.

The author guide promises automatic retesting after the base advances and documents batch priority (`site/content/en/docs/components/core/tide/pr-authors.md:37-43`). A scheduling fix must preserve required-test gating and useful batch throughput.

### Proposed approach

**Recommended: failure-aware serial fallback.** First capture a multi-sync reproduction, then distinguish an actually failed required batch job from missing jobs. Carry a valid failed-batch signal from accumulation into scheduling, scoped to the same base and PR heads. When the pool would otherwise retry an unchanged failed combination indefinitely, give a missing individual retest a chance to run; preserve the normal merge-success path and do not merge stale individual greens. Once that serial result passes, normal merging can advance the base.

- Trade-offs: extra individual work and reduced batch throughput on transient failures, in exchange for progress; define behavior with simultaneous pending and failed batches, changed config, and garbage-collected jobs.

- Backwards compatibility: no public API/config change is inherently needed, but deployment-visible retry ordering changes. Do not reject all stale-green batch candidates: they are precisely what normal batching retests.

- Testing needed in implementation work: repeated fast-failing two-PR batches at a stable base; serial fallback then merge; ordinary pending-batch serial retest; mixed pending/failed contexts; missing `run_before_merge` jobs; old failures with newer successful results; base/head changes; `run_if_changed`; no required-test bypass.

**Alternative: shrink failed batches.** PR #538 removes the highest-numbered PR in a failed set and selects another batch, falling through to serial work for a two-PR pool. This may retain throughput for larger pools, but is not binary bisection and needs explicit convergence/fairness rules. Its helper treats newest as lowest priority without considering configured label priority; multiple failed sets are assigned from map iteration, and missing-job states are lumped into failures. Its latest diff adds a synthetic three-PR shrink case, not a demonstrated multi-sync progress guarantee. Treat it as a design starting point, not a ready or merged fix.

## Effort assessment

**Effort Level**: 3 - Large

The deciding factors are unresolved incident conditions and a deployment-visible change to Tide's core scheduling policy. A small patch still requires expertise to preserve merge safety and progress across repeated syncs; the existing unmerged PR does not reduce this level.

- **Solution certainty**: significant uncertainty — conditional starvation is clear, but production causality and fallback versus shrinking semantics need agreement.

- **Blast radius**: core paths and deployment-visible behavior — batch retry ordering, serial progress, test reuse and required-context handling across deployments.

- **Expertise**: Tide result accumulation, job indexing/lifetime, GitHub contexts, batching and priority semantics.

- **Scope estimate**: Medium — likely `pkg/tide` accumulation/scheduling and multi-sync regression coverage; no public configuration change is inherently required.

**Recommended labels**: `kind/bug` (reported loss of progress), `area/tide` (scheduler ownership). Both are already present. Neither `good-first-issue` nor `help-needed` is appropriate at level 3. No priority label is justified without current impact/scope evidence.

**Guidance for contributors**: Read accumulation, eligibility, `takeAction` and the existing retry/pending-batch tests first. Agree a progress invariant and failure classification with Tide maintainers before implementation. Use #538 as prior art, including its multi-failure, priority and missing-job limitations.

## Briefing summary

Completed the seven-slide interactive briefing: issue overview, legitimacy, conditional root cause, technical analysis, solution approaches, effort and recommendations. Keep #474 open as a legitimate Tide scheduling report. The code permits repeated failed-batch selection under specific conditions, but individual retesting can run while a batch is pending, so the production cause remains unverified. PR #538 closed unmerged; #1041 was withdrawn and is not independent confirmation. Recommended collecting a multi-sync reproduction before agreeing failure-aware serial fallback versus batch shrinking. Effort is level 3 because of uncertainty and deployment-visible changes to core scheduling; retain `kind/bug` and `area/tide` without newcomer or priority labels.

Briefed maintainer on: 2026-10-28T01:17:50Z

Key questions asked:

- No substantive questions were asked; the maintainer chose to continue through all seven slides.

Maintainer decision:

- No implementation, label, or publication decision was indicated. The recommendations above remain proposed actions.

## Findings

### [cause] Conditional starvation of individual retesting

- detail: With stale individual results, at least two batch-eligible PRs, and no batch classified pending or successful at each sync, `takeAction` can repeatedly select the same batch before individual retesting. This is a code-supported hypothesis, not a reproduced explanation of the incidents; a pending batch can allow individual retesting.

- evidence: `pkg/tide/tide.go:1615-1665`; [March 10 qualification](https://github.com/kubernetes-sigs/prow/issues/474#issuecomment-4027955562).

### [related-code] Batch accumulation omits invalid batches

- where: `pkg/tide/tide.go:1082-1113`

- excerpt: |
    case pendingState:
        pendingBatch = state.prs
    case successState:
        successBatch = state.prs

- detail: Both missing required jobs and actual failures produce `failureState`; a fix must distinguish them.

### [related-code] Stale successes remain eligible for batch retesting

- where: `pkg/tide/tide.go:1116-1210,1271-1381`

- excerpt: |
    if c.isRetestEligible(sp.log, &pr, cc[pr.Number]) {
        candidates = append(candidates, pr)
    }

- detail: Batch retest eligibility accepts successful head contexts regardless of base. Individual success classification requires current-base jobs or the skip-retest sentinel. That distinction enables ordinary batching and is not itself a defect.

### [related-code] Completed unsuccessful batch jobs can retry

- where: `pkg/tide/tide.go:1557-1612,2259-2287`

- excerpt: |
    if pj.Complete() && pj.Status.State != prowapi.SuccessState {
        return nil
    }

- detail: Such jobs are excluded from the deduplication index. Nonfailed batch jobs can be reused, so repeated `TRIGGER_BATCH` actions do not prove that every job restarted.

### [reproducibility] Incident reports do not yet establish the full loop

- detail: Production reports and PR #538's log excerpt support investigating repeated batches. Fast-failure reproduction and why pending-batch snapshots showed WAIT remain unverified; no runtime reproduction was executed.

- evidence: [Incident log](https://github.com/kubernetes-sigs/prow/pull/538#issuecomment-3897386597); [stale-base hypothesis](https://github.com/kubernetes-sigs/prow/issues/474#issuecomment-4020200695).

### [related-pr] Closed batch-shrinking proposal

- ref: kubernetes-sigs/prow#538

- relevance: CLOSED, unmerged on 2026-10-17. Proposes removing the highest-numbered PR from a failed batch; prior art requiring convergence, priority, and missing-job handling review.

### [related-issue] Withdrawn overlapping report

- ref: kubernetes-sigs/prow#1041

- relevance: CLOSED. Describes the stale-base/repeated-batch hypothesis; the author withdrew its AI analysis on 2026-10-27, so it is not independent confirmation.

### [related-issue] Superseded batch jobs continue running

- ref: kubernetes-sigs/prow#650

- relevance: OPEN. Concerns obsolete batch jobs after the base moves; related scheduling area, different trigger.

### [related-issue] GitHub review protection blocked merge attempts

- ref: kubernetes-sigs/prow#673

- relevance: CLOSED. Solved through configuration introduced in #579; distinct from batch test failures.

### [related-issue] Priority merge feature request

- ref: kubernetes-sigs/prow#506

- relevance: CLOSED. Requests expedited individual merging, not recovery from this failed-batch loop.

### [related-pr] Merge errors surfaced in history

- ref: kubernetes-sigs/prow#643

- relevance: MERGED on 2026-03-09. Improves observability of merge errors; does not fix failed-batch scheduling.

## Checked

- Read full issue and discussion, including the March 10 qualification and October 28 reopening.

- Queried cross-reference timeline; found #506, #538, #643.

- Searched all issues using `tide batch`, `batch fail`, and `merge incompatible`; opened relevant candidates #1041, #673, #650 and #506.

- No confirmed older canonical duplicate found. #1041 is newer and withdrawn; #673 has a different cause.

- Read PR #538 diff and nearby tests; no implementation or runtime reproduction performed.

- Ruled out the previous blanket claim that serial retesting is always starved; the pending-batch path can trigger it.

- No recent regression commit identified by focused action-order/failure-state history search.

- Analysis inspected code and tests without running tests or changing implementation. `/triage:save` generates `TRIAGE.html` from this Markdown.

## Next steps

- Capture full sync buckets plus job/context timestamps at a stable base and reproduce the failure across successive syncs.

- Ask whether PR #538 should be revived or replaced after the fallback policy is agreed with Tide maintainers.

- Retain `kind/bug` and `area/tide`; do not add newcomer or priority labels on present evidence.

## Advice

No change since triage at 2026-10-28T01:17:50Z: #474 remains open with `kind/bug` and `area/tide`, no assignee or milestone, and no new comments or cross-references. PR #538 remains closed and unmerged; #643 merged only an observability improvement. No active linked fix was identified. The existing next steps still apply.

### Suggested action: request the evidence needed for reproduction

The unresolved finding is why individual retesting did not rescue the pool. Ask for complete successive sync states before choosing a scheduling change; the latest human comment only says the issue remains. Suggested command (not executed):

```bash
gh issue comment 474 --repo kubernetes-sigs/prow --body '> *This was generated by AI during triage.*

@bn222 @Prucek could you share a recent occurrence with successive Tide sync logs showing prs-passing, prs-pending, prs-missing, batch-passing, batch-pending, actions and targets? Please include job/context timestamps, base and PR-head SHAs, Tide revision, sync period, and relevant job/batching configuration.

Tide can trigger an individual retest while a batch is pending. We need to understand why that did not make progress here, including whether a required batch job failed before the next sync. PR #538 closed unmerged; a reproduction would help decide whether to revive that approach or use individual retesting after batch failure.'
```

Keep the issue open and retain its existing labels; no label edit or fresh PR link is needed. The level-3 assessment does not justify newcomer labels, and there is no evidence for a priority, milestone, or assignment recommendation. Decide whether to revive or replace #538 after reproduction and maintainer agreement on fallback behavior.

## Open questions

- Could you provide successive sync logs including `prs-passing`, `prs-pending`, `prs-missing`, `batch-passing`, `batch-pending`, actions and targets, so we can determine what prevented an individual retest while the recorded batch remained pending?

- Does the incident reproduce when a required batch job fails before the next Tide sync? Please include job/context timestamps, base and PR-head SHAs, Tide revision, sync period and relevant job/batching configuration.
