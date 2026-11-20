---
issue: kubernetes-sigs/prow#1078
title: "Stale queued presubmits run after the base advances"
state: open
labels: ""
main_sha: 9c8eff7c98cc979e833369aefe5d1e86afcf063f
triaged_at: 2026-11-20T00:36:57Z
verdict: needs-discussion
---

# Triage

## Verdict

Needs discussion: Prow works as designed here; the only defect is that a queued job spends a run
that cannot count, because the base moved before it started.

A presubmit captures its base SHA when the PR event fires and never re-evaluates it. A job held
behind `max_concurrency` therefore runs pre-move base + current PR head. That is the same outcome as
a base advance during a run, and Tide's base-SHA validity rule means the result is not evidence for
the new base, so Tide re-triggers the job for pooled PRs. The maintainer's reading (2026-11-20) is
that this is intended behavior and a mild inefficiency rather than a bug; queueing differs from a
running base advance only by the pre-flight window, where something smarter could be done. Disposition
(keep as a backlog enhancement vs. close `wontfix` with documentation) is open.

## What the issue reports

- A presubmit that waits in Prow's queue starts with the base SHA from its creation, so it tests stale
  base + current PR head; the reporter's example ran 4h41m after creation and ~3h40m after another PR
  merged to `main`.
- The reporter argues the run "can't vote, nor present the fresh state", so the PR needs at least one
  more run before it can merge.
- Requested remedy: stale queued jobs "should be cancelled or superseded / auto rebased".
- Scope is presubmits only ("non tide batch jobs"); Tide batch staleness is tracked separately.
- The run was created by the trigger plugin, not Tide: labels show `created-by-prow` and
  `event-GUID`, and no `created-by-tide`.

## Findings

### [reproducibility] stale base confirmed from the ProwJob artifacts
- detail: ProwJob created 2026-11-18T14:19:47Z with `prow.k8s.io/is-optional: "false"` and no
  `created-by-tide`; `started.json` refs are `main:e9b962f598371bf9d002d94536ff0c32eff8fc08,18626:4cca9de8...`,
  so the pod checked out the pre-merge base. Status: `pendingTime` 19:00:31Z, `completionTime`
  20:34:54Z, state `success`.
- evidence: `storage.googleapis.com/kubevirt-prow/pr-logs/pull/kubevirt_kubevirt/18626/pull-kubevirt-e2e-k8s-1.37-sig-compute-serial/2123129241695424512/{prowjob,started}.json`

### [reproducibility] the delay is a per-job concurrency limit, not a Prow queue
- detail: spec has `max_concurrency: 11` and no `job_queue_name`. Prow has no presubmit queue unless
  a concurrency limit or `job_queue_name` is configured; the job sat in `TriggeredState` while Plank
  waited for a slot, so it is not a pod-scheduling delay.
- evidence: example prowjob.json; `pkg/plank/reconciler.go:905-975`

### [cause] base SHA is captured once, at event time
- detail: the trigger plugin resolves `heads/<base ref>` on the PR event and freezes it into
  `refs.base_sha`; nothing re-reads the branch while the ProwJob waits.
- evidence: `pkg/plugins/trigger/pull-request.go:52-59`, `pkg/pjutil/pjutil.go:130-150`

### [cause] supersede requires a newer run to already exist
- detail: `terminateDupes` aborts older pending/triggered runs of the same job and PR set; the digest
  excludes commit SHAs, so a base advance alone supersedes nothing. Replacements come from Tide (pool
  PRs only), a new PR event, or `/test`.
- evidence: `pkg/plank/reconciler.go:438-450`, `pkg/pjutil/abort.go:61-118`

### [cause] Tide retests only PRs that are in its pool
- detail: subpools contain only PRs matching `tide.queries` with required labels. kubevirt runs Tide
  (`sync_period: 2m`), but PR #18626 was not merge-eligible, so no fresh run was created and
  `terminateDupes` had no newer run to prefer.
- evidence: `pkg/tide/tide.go:642-662`

### [cause] queueing differs from a running base advance only by the pre-flight window
- detail: both leave a run pinned to an old base with that base recorded; queueing adds an
  arbitrarily long wait plus the option to decide before spending the run.
- evidence: `pkg/pod-utils/clone/clone.go:266-300`, `pkg/config/config.go:3533-3552`

### [cause] a stale green keeps the PR pooled, so Tide re-triggers and the PR is not stuck
- detail: `filterPR` judges required contexts by state only, so green keeps the PR in the pool;
  `prowJobsFromContexts` drops it as evidence on the base mismatch, `accumulate` records the job as
  missing, and `takeAction` triggers a fresh run. A stale red behaves like any failing required
  context (push or `/test`), not as a distinct failure mode.
- evidence: `pkg/tide/tide.go:825-860`, `:935-959`, `:1110-1145`, `:1183-1184`, `:1653-1655`, `:1328-1381`

### [related-code] clonerefs pins the checkout to BaseSHA
- where: `pkg/pod-utils/clone/clone.go:266-300`
- excerpt: |
    if refs.BaseSHA != "" {
        fetchRef = refs.BaseSHA
        target = refs.BaseSHA
    } else {
        fetchRef = refs.BaseRef
        target = "FETCH_HEAD"
    }
    ...
    commands = append(commands, g.gitCommand("checkout", target))

### [related-code] the reported status carries the base SHA
- where: `pkg/github/report/report.go:84-94`
- excerpt: |
    Description: config.ContextDescriptionWithBaseSha(pj.Status.Description, refs.BaseSHA),

### [related-code] Tide lists ProwJobs by base SHA
- where: `pkg/tide/tide.go:2043-2056`, key at `:2232-2234`
- excerpt: |
    ctrlruntimeclient.MatchingFields{cacheIndexName: cacheIndexKey(sp.org, sp.repo, sp.branch, sp.sha)}

### [related-code] Plank has no GitHub client today
- where: `cmd/prow-controller-manager/main.go:65,88`
- excerpt: |
    github prowflagutil.GitHubOptions // TODO(fejta): remove

### [related-pr] PR #1080 is the WIP implementation
- ref: kubernetes-sigs/prow#1080
- relevance: refreshes `refs.base_sha` in Plank before `startPod`; review raised job-definition
  drift, Gerrit presubmits failing the GitHub lookup, and pinned runs (`mkpj --base-sha`, Deck rerun)
  being silently re-based.

### [related-issue] #650 and PR #744 are the Tide batch analog
- ref: kubernetes-sigs/prow#650, kubernetes-sigs/prow#744
- relevance: obsolete batch ProwJobs not aborted when superseded; same family, different job type.

### [related-issue] #986 covers a neighbouring retest gap
- ref: kubernetes-sigs/prow#986
- relevance: manually triggered required presubmits not retested by Tide when the base moves.

### [related-issue] #1041 was a base-SHA retest livelock
- ref: kubernetes-sigs/prow#1041
- relevance: closed; shows how delicate Tide's base-SHA retest decisions are.

## Resolved

### [cause] "Tide will ignore the stale result"
- detail: withdrawn as overstatement. The green is not evidence for the new base, but it is not
  discarded: it keeps the PR pooled and Tide re-triggers. See the stale-green finding above.
- evidence: `pkg/tide/tide.go:1110-1145`, `:1653-1655`

### [cause] "a stale red strands the PR"
- detail: withdrawn. A stale red blocks like any failing required context and is resolved by a push
  or `/test`; it is not a separate failure mode.
- evidence: `pkg/tide/tide.go:825-860`

## Checked

- `gh api repos/kubernetes-sigs/prow/issues/1078/timeline --paginate` — the only cross-reference is PR #1080.
- `gh issue list --search` for "stale presubmit", "base sha queued", "presubmit base", "queued job base" — no earlier report of stale queued presubmits.
- Ruled out a Tide merge-correctness bug: stale runs cannot add passing evidence (base-keyed index and description check).
- Ruled out batch/Tide jobs: #650/#744 are explicitly batch-only; the issue is presubmit-scoped.
- Ruled out clonerefs as a culprit: the base pin is intentional and deterministic.
- Ruled out pod scheduling: `pendingTime` proves Plank held the ProwJob, matching `max_concurrency: 11`.
- Confirmed there is no runtime precedent for mutating `spec.refs.base_sha`; all assignments are at creation or in Tide batch creation.
- Confirmed Plank has no GitHub client today, only leftover `GitHubOptions` flags.
- Checked kubevirt's Prow config in `kubevirt/project-infra`: Tide is enabled for `kubevirt/kubevirt` (`sync_period: 2m`, `batch_size_limit: 8`).
- Verified the artifacts directly instead of trusting the issue's evidence table.
- `gh api repos/kubernetes-sigs/prow/issues/comments/7002427182` is a bot `ok-to-test` comment on unrelated PR #1081, not relevant to this issue.

## Next steps

- Decide disposition with the maintainer: keep as `kind/feature` + `priority/backlog`, or close `wontfix` with a documentation note.
- If kept: decide cancel-on-base-push vs. Plank refresh, and tell PR #1080 to wait or narrow to the agreed direction.
- Ask the reporter whether the pre-flight saving justifies the complexity, given that a base advance during a running job has the same outcome and no remedy.
- Link #986 and #650/#744 from the issue if the maintainer wants the relationships recorded.
- If `wontfix`: document that a queued or concurrency-limited presubmit pins its base SHA and that Tide re-runs it against the current base.

## Open questions

- Is the pre-flight window worth acting on at all, given that a base advance during a running job produces the same stale result and is not treated as a bug?
- If it is: cancel or re-base? Cancelling needs coverage for PRs outside Tide's pool; re-basing needs a guarantee that the job definition still matches the new base.
- Should pinned runs (`mkpj --base-sha`, Deck's original-spec rerun) be excluded explicitly if a refresh is ever implemented?
- Should the pinning behavior be documented in user-facing docs, independent of the disposition?
